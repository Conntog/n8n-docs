# Esmer (ESMO) API Integration Specification: Tasks & Project Management

> **Category:** 03 - Tasks & Project Management
> **Version:** 1.0.0
> **Last Updated:** 2026-02-09
> **Status:** Draft

---

## Table of Contents

- [Cross-Service Comparison Matrix](#cross-service-comparison-matrix)
- [1. Google Tasks (Google Tasks API v1)](#1-google-tasks-google-tasks-api-v1)
- [2. Todoist (Todoist REST API v2)](#2-todoist-todoist-rest-api-v2)
- [3. Microsoft To Do (Microsoft Graph API)](#3-microsoft-to-do-microsoft-graph-api)
- [4. Notion (Notion API)](#4-notion-notion-api)
- [5. Asana (Asana REST API)](#5-asana-asana-rest-api)
- [6. ClickUp (ClickUp API v2)](#6-clickup-clickup-api-v2)
- [7. Linear (Linear GraphQL API)](#7-linear-linear-graphql-api)
- [8. Trello (Trello REST API)](#8-trello-trello-rest-api)
- [9. Jira Software (Atlassian REST API v3)](#9-jira-software-atlassian-rest-api-v3)
- [10. monday.com (monday.com GraphQL API)](#10-mondaycom-mondaycom-graphql-api)

---

## Cross-Service Comparison Matrix

| Feature | Google Tasks | Todoist | Microsoft To Do | Notion | Asana | ClickUp | Linear | Trello | Jira | monday.com |
|---|---|---|---|---|---|---|---|---|---|---|
| **API Style** | REST | REST | REST (Graph) | REST | REST | REST | GraphQL | REST | REST | GraphQL |
| **Auth Method** | OAuth 2.0 | OAuth 2.0 / API Key | OAuth 2.0 | OAuth 2.0 / Internal Token | OAuth 2.0 / PAT | OAuth 2.0 / API Token | OAuth 2.0 / API Key | API Key + Token | API Token / Basic | OAuth 2.0 / API Token |
| **Rate Limit** | Per-user quota | 450 req/min | 10,000 req/10min | 3 req/sec avg | 1,500 req/min | 100 req/min | 1,500 complexity/min | 100 req/10sec/key | Rate headers | 5M complexity/min |
| **Webhooks** | No | Yes | Change notifications (subscriptions) | No (polling) | Yes | Yes | Yes | Yes (via webhooks) | Yes (webhooks) | Yes |
| **Subtasks** | No | Yes (sub-tasks) | No | Via relations | Yes | Yes (subtasks + checklists) | Yes (sub-issues) | Yes (checklists) | Yes (sub-tasks) | Yes (subitems) |
| **Attachments** | No | No (via comments) | File links (linked resources) | File blocks | Yes | Yes | Yes (via comments) | Yes | Yes | Yes (file columns) |
| **Labels/Tags** | No | Yes (labels) | No (categories) | Multi-select properties | Yes (tags) | Yes (tags) | Yes (labels) | Yes (labels) | Yes (labels) | Yes (tags column) |
| **Projects/Boards** | Task Lists | Projects + Sections | Lists | Databases | Projects + Sections | Spaces/Folders/Lists | Teams/Projects | Boards/Lists | Projects | Boards/Groups |
| **Comments** | No | Yes | No | Yes (blocks) | Yes | Yes | Yes | Yes | Yes | Yes (updates) |
| **Time Tracking** | No | No | No | Via properties | No (third-party) | Yes | Yes (estimates) | No (Power-Ups) | Yes (via plugins) | Yes (time column) |
| **Custom Fields** | No | No | No | Yes (properties) | Yes | Yes | Yes | Yes (Custom Fields) | Yes | Yes (columns) |
| **Mobile Suitability** | High | High | High | Medium | Medium | Medium | Medium | High | Low-Medium | Medium |
| **Complexity** | Very Low | Low | Low | Medium | Medium | High | Medium | Medium | High | Medium |
| **Best For Esmer** | Simple personal tasks | Personal task delegation | Microsoft ecosystem users | Knowledge + task DB | Team project management | Complex project workflows | Engineering issue tracking | Visual board management | Enterprise issue tracking | Team work management |

---

## 1. Google Tasks (Google Tasks API v1)

### 1.1 Service Overview

Google Tasks is a lightweight task management service integrated into Google Workspace (Gmail, Google Calendar). It supports simple task lists with due dates and notes. Esmer will use Google Tasks for users who want quick, no-friction task creation and management directly tied to their Google account. Ideal for personal task delegation where the user says "remind me to..." or "add a task to..." and Esmer creates it instantly.

### 1.2 Authentication

**Method:** OAuth 2.0

| Parameter | Value |
|---|---|
| Authorization URL | `https://accounts.google.com/o/oauth2/v2/auth` |
| Token URL | `https://oauth2.googleapis.com/token` |
| Grant Type | `authorization_code` |
| Refresh Token Support | Yes (with `access_type=offline`) |

**Required Scopes:**

| Scope | Description | Required |
|---|---|---|
| `https://www.googleapis.com/auth/tasks` | Full access to tasks and task lists (read/write/delete) | Yes |
| `https://www.googleapis.com/auth/tasks.readonly` | Read-only access to tasks and task lists | Optional (use for read-only mode) |

**Recommendations for Esmer:**
- Use OAuth 2.0 with `access_type=offline` and `prompt=consent` to obtain a refresh token on first authorization.
- Request the full `tasks` scope since Esmer needs to create and update tasks.
- Google Tasks does not support Service Accounts; OAuth 2.0 is the only path.
- Store refresh tokens securely; access tokens expire after 3600 seconds.
- Implement the Google OAuth consent flow using an in-app browser or system browser redirect on mobile.

### 1.3 Base URL

```
https://www.googleapis.com/tasks/v1
```

### 1.4 Rate Limits

| Limit | Value |
|---|---|
| Per-user per-project quota | ~50,000 queries/day (default) |
| Per-user rate limit | ~500 queries/100 seconds/user |
| Per-project rate limit | ~50,000 queries/100 seconds/project |

Rate limit errors return HTTP `403` or `429`. Implement exponential backoff. Check the `Retry-After` header when present.

### 1.5 API Endpoints

#### 1.5.1 Task Lists

##### List all task lists

```
GET /users/@me/lists
```

**Description:** Returns all the authenticated user's task lists.

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `maxResults` | integer | No | Maximum number of task lists to return (default 20, max 100) |
| `pageToken` | string | No | Token for pagination |

**Response:**

```json
{
  "kind": "tasks#taskLists",
  "etag": "string",
  "nextPageToken": "string",
  "items": [
    {
      "kind": "tasks#taskList",
      "id": "MTYzMTkwNjI0NTY4NjEwOTk3OTM6MDow",
      "etag": "string",
      "title": "My Tasks",
      "updated": "2026-02-09T12:00:00.000Z",
      "selfLink": "https://www.googleapis.com/tasks/v1/users/@me/lists/MTYz..."
    }
  ]
}
```

**Error Codes:**

| Code | Description |
|---|---|
| 401 | Invalid or expired access token |
| 403 | Rate limit exceeded or insufficient permissions |
| 404 | Resource not found |

##### Get a task list

```
GET /users/@me/lists/{tasklist}
```

**Description:** Returns the authenticated user's specified task list.

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `tasklist` | string | Yes | Task list identifier |

**Response:**

```json
{
  "kind": "tasks#taskList",
  "id": "MTYzMTkwNjI0NTY4NjEwOTk3OTM6MDow",
  "etag": "string",
  "title": "My Tasks",
  "updated": "2026-02-09T12:00:00.000Z",
  "selfLink": "string"
}
```

##### Create a task list

```
POST /users/@me/lists
```

**Description:** Creates a new task list.

**Request Body:**

```json
{
  "title": "Work Tasks"
}
```

**Response:** Returns the created `taskList` resource (same schema as Get).

##### Update a task list

```
PATCH /users/@me/lists/{tasklist}
```

**Description:** Updates the authenticated user's specified task list.

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `tasklist` | string | Yes | Task list identifier |

**Request Body:**

```json
{
  "title": "Updated Title"
}
```

**Response:** Returns the updated `taskList` resource.

##### Delete a task list

```
DELETE /users/@me/lists/{tasklist}
```

**Description:** Deletes the authenticated user's specified task list.

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `tasklist` | string | Yes | Task list identifier |

**Response:** Empty body with `204 No Content`.

#### 1.5.2 Tasks

##### List all tasks

```
GET /lists/{tasklist}/tasks
```

**Description:** Returns all tasks in the specified task list.

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `tasklist` | string | Yes | Task list identifier |

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `completedMax` | string (datetime) | No | Upper bound for task completion date (RFC 3339) |
| `completedMin` | string (datetime) | No | Lower bound for task completion date (RFC 3339) |
| `dueMax` | string (datetime) | No | Upper bound for task due date (RFC 3339) |
| `dueMin` | string (datetime) | No | Lower bound for task due date (RFC 3339) |
| `maxResults` | integer | No | Maximum number of tasks to return (default 20, max 100) |
| `pageToken` | string | No | Token for pagination |
| `showCompleted` | boolean | No | Whether to include completed tasks (default true) |
| `showDeleted` | boolean | No | Whether to include deleted tasks (default false) |
| `showHidden` | boolean | No | Whether to include hidden tasks (default false) |
| `updatedMin` | string (datetime) | No | Lower bound for last modification time (RFC 3339) |

**Response:**

```json
{
  "kind": "tasks#tasks",
  "etag": "string",
  "nextPageToken": "string",
  "items": [
    {
      "kind": "tasks#task",
      "id": "dGFzazEyMzQ1",
      "etag": "string",
      "title": "Buy groceries",
      "updated": "2026-02-09T12:00:00.000Z",
      "selfLink": "string",
      "parent": "string",
      "position": "00000000000000000001",
      "notes": "Milk, eggs, bread",
      "status": "needsAction",
      "due": "2026-02-10T00:00:00.000Z",
      "completed": null,
      "deleted": false,
      "hidden": false,
      "links": [
        {
          "type": "email",
          "description": "string",
          "link": "string"
        }
      ]
    }
  ]
}
```

##### Get a task

```
GET /lists/{tasklist}/tasks/{task}
```

**Description:** Returns the specified task.

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `tasklist` | string | Yes | Task list identifier |
| `task` | string | Yes | Task identifier |

**Response:** Returns a single `task` resource (same schema as items in list).

##### Create a task

```
POST /lists/{tasklist}/tasks
```

**Description:** Creates a new task on the specified task list.

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `tasklist` | string | Yes | Task list identifier |

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `parent` | string | No | Parent task identifier (for subtask-like positioning) |
| `previous` | string | No | Previous sibling task identifier (for ordering) |

**Request Body:**

```json
{
  "title": "Buy groceries",
  "notes": "Milk, eggs, bread",
  "due": "2026-02-10T00:00:00.000Z",
  "status": "needsAction"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `title` | string | Yes | Title of the task |
| `notes` | string | No | Notes describing the task (max 8192 chars) |
| `due` | string (datetime) | No | Due date in RFC 3339 format |
| `status` | string | No | Status: `needsAction` or `completed` |

**Response:** Returns the created `task` resource.

##### Update a task

```
PATCH /lists/{tasklist}/tasks/{task}
```

**Description:** Updates the specified task.

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `tasklist` | string | Yes | Task list identifier |
| `task` | string | Yes | Task identifier |

**Request Body:** Same fields as Create (all optional for PATCH).

```json
{
  "title": "Buy groceries (updated)",
  "status": "completed"
}
```

**Response:** Returns the updated `task` resource.

##### Delete a task

```
DELETE /lists/{tasklist}/tasks/{task}
```

**Description:** Deletes the specified task from the task list.

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `tasklist` | string | Yes | Task list identifier |
| `task` | string | Yes | Task identifier |

**Response:** Empty body with `204 No Content`.

##### Move a task

```
POST /lists/{tasklist}/tasks/{task}/move
```

**Description:** Moves the specified task to another position in the task list.

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `tasklist` | string | Yes | Task list identifier |
| `task` | string | Yes | Task identifier |

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `parent` | string | No | New parent task identifier (empty string to move to top level) |
| `previous` | string | No | New previous sibling task identifier |

**Response:** Returns the moved `task` resource.

##### Clear completed tasks

```
POST /lists/{tasklist}/clear
```

**Description:** Clears all completed tasks from the specified task list.

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `tasklist` | string | Yes | Task list identifier |

**Response:** Empty body with `204 No Content`.

### 1.6 Webhooks / Real-time

Google Tasks API does **not** support webhooks or push notifications natively. Esmer must use polling to detect changes.

**Polling Strategy for Esmer:**
- Use the `updatedMin` query parameter on the List Tasks endpoint to fetch only tasks modified since the last sync.
- Poll every 60-120 seconds for active users; reduce frequency for background sync.
- Store the `etag` of each task and task list to detect changes efficiently.

### 1.7 Error Handling

| HTTP Code | Error | Description | Esmer Action |
|---|---|---|---|
| 400 | `invalid` | Invalid request body or parameters | Validate inputs before sending |
| 401 | `authError` | Invalid, expired, or revoked token | Refresh access token; if refresh fails, re-authenticate |
| 403 | `dailyLimitExceeded` | Daily quota exceeded | Back off until quota resets at midnight PT |
| 403 | `rateLimitExceeded` | Per-user rate limit exceeded | Exponential backoff with jitter |
| 403 | `forbidden` | Insufficient permissions | Check scopes; re-request authorization |
| 404 | `notFound` | Task or task list not found | Show user-friendly "not found" message |
| 429 | `rateLimitExceeded` | Too many requests | Respect `Retry-After` header; exponential backoff |
| 500 | `backendError` | Google server error | Retry with exponential backoff (max 3 retries) |
| 503 | `backendError` | Service temporarily unavailable | Retry with exponential backoff |

### 1.8 Mobile-Specific Notes

- **Lightweight API:** Google Tasks has an extremely simple data model (title, notes, due date, status). Perfect for quick voice-driven task creation.
- **Offline Support:** Cache task lists and tasks locally. Queue create/update/delete operations when offline and sync when connectivity returns. Use `etag` for conflict resolution.
- **Pagination:** Default page size is 20. For mobile, fetch one page at a time and implement infinite scroll.
- **Deep Linking:** Tasks created via the API appear in Google Tasks within Gmail and Google Calendar. No deep link URL scheme is available for the standalone Tasks app.
- **Token Storage:** Store OAuth refresh tokens in the device keychain (iOS Keychain / Android Keystore). Access tokens should be kept in memory only.
- **Minimal Payload:** Responses are small (typically under 5KB per page). Suitable for low-bandwidth mobile scenarios.

---

## 2. Todoist (Todoist REST API v2)

### 2.1 Service Overview

Todoist is a popular personal and team task management application with natural language date parsing, projects, labels, priorities, and filters. Esmer will use Todoist for users who rely on it as their primary task manager. Esmer can create tasks from natural language commands, manage projects, and close/reopen tasks on behalf of the user.

### 2.2 Authentication

**Methods:** OAuth 2.0, API Key (Personal Token)

| Parameter | Value |
|---|---|
| Authorization URL | `https://todoist.com/oauth/authorize` |
| Token URL | `https://todoist.com/oauth/access_token` |
| Grant Type | `authorization_code` |
| Refresh Token Support | No (tokens do not expire unless revoked) |

**Required Scopes:**

| Scope | Description | Required |
|---|---|---|
| `data:read` | Read access to all user data | Yes |
| `data:read_write` | Read and write access to all user data | Yes (for task creation/update) |
| `data:delete` | Delete access to user data | Yes (for task deletion) |
| `project:delete` | Delete projects | Optional |
| `task:add` | Add tasks only (minimal scope) | Optional (subset of `data:read_write`) |

**API Key Alternative:**
- Users can retrieve a personal API token from Todoist Settings > Integrations > Developer > API Token.
- Pass via `Authorization: Bearer {token}` header.
- Simpler for personal use but not suitable for multi-user Esmer deployments.

**Recommendations for Esmer:**
- Use OAuth 2.0 for multi-user support. Todoist access tokens do not expire, so no refresh logic is needed.
- Request `data:read_write` and `data:delete` scopes for full task management capability.
- For quick prototyping, the API Key method works but cannot be used for delegated access.

### 2.3 Base URL

```
https://api.todoist.com/rest/v2
```

Sync API (for batch operations):
```
https://api.todoist.com/sync/v9
```

### 2.4 Rate Limits

| Limit | Value |
|---|---|
| REST API | 450 requests per minute per user |
| Sync API | 450 requests per minute per user |

Rate limit responses return HTTP `429`. The response includes a `Retry-After` header indicating seconds to wait.

### 2.5 API Endpoints

#### 2.5.1 Projects

##### Get all projects

```
GET /projects
```

**Description:** Returns all user projects.

**Headers:**

| Name | Value | Required |
|---|---|---|
| `Authorization` | `Bearer {token}` | Yes |

**Response:**

```json
[
  {
    "id": "2203306141",
    "name": "Inbox",
    "comment_count": 0,
    "order": 0,
    "color": "charcoal",
    "is_shared": false,
    "is_favorite": false,
    "is_inbox_project": true,
    "is_team_inbox": false,
    "view_style": "list",
    "url": "https://todoist.com/showProject?id=2203306141",
    "parent_id": null
  }
]
```

##### Get a project

```
GET /projects/{project_id}
```

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `project_id` | string | Yes | The project ID |

**Response:** Single project object (same schema as above).

##### Create a project

```
POST /projects
```

**Request Body:**

```json
{
  "name": "Work",
  "color": "blue",
  "is_favorite": true,
  "parent_id": null,
  "view_style": "list"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Name of the project |
| `color` | string | No | Color name (e.g., `berry_red`, `blue`, `charcoal`) |
| `is_favorite` | boolean | No | Whether the project is a favorite |
| `parent_id` | string | No | Parent project ID for sub-projects |
| `view_style` | string | No | `list` or `board` |

**Response:** Created project object.

##### Update a project

```
POST /projects/{project_id}
```

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `project_id` | string | Yes | The project ID |

**Request Body:** Same fields as create (all optional).

**Response:** Updated project object.

##### Delete a project

```
DELETE /projects/{project_id}
```

**Response:** `204 No Content`.

#### 2.5.2 Sections

##### Get all sections

```
GET /sections
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `project_id` | string | No | Filter sections by project |

**Response:**

```json
[
  {
    "id": "7025",
    "project_id": "2203306141",
    "order": 1,
    "name": "Groceries"
  }
]
```

##### Create a section

```
POST /sections
```

**Request Body:**

```json
{
  "name": "Groceries",
  "project_id": "2203306141",
  "order": 1
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Section name |
| `project_id` | string | Yes | Project the section belongs to |
| `order` | integer | No | Order within the project |

##### Update a section

```
POST /sections/{section_id}
```

**Request Body:**

```json
{
  "name": "Updated Section Name"
}
```

##### Delete a section

```
DELETE /sections/{section_id}
```

**Response:** `204 No Content`.

#### 2.5.3 Tasks

##### Get all tasks

```
GET /tasks
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `project_id` | string | No | Filter by project |
| `section_id` | string | No | Filter by section |
| `label` | string | No | Filter by label name |
| `filter` | string | No | Todoist filter query (e.g., `today`, `overdue`, `p1`) |
| `lang` | string | No | IETF language tag for due date parsing |
| `ids` | string | No | Comma-separated list of task IDs (max 100) |

**Response:**

```json
[
  {
    "id": "2995104339",
    "assigner_id": null,
    "assignee_id": null,
    "project_id": "2203306141",
    "section_id": null,
    "parent_id": null,
    "order": 1,
    "content": "Buy groceries",
    "description": "Milk, eggs, bread",
    "is_completed": false,
    "labels": ["shopping"],
    "priority": 4,
    "comment_count": 0,
    "creator_id": "2671355",
    "created_at": "2026-02-09T12:00:00.000000Z",
    "due": {
      "date": "2026-02-10",
      "string": "tomorrow",
      "lang": "en",
      "is_recurring": false,
      "datetime": "2026-02-10T14:00:00.000000Z",
      "timezone": "America/New_York"
    },
    "url": "https://todoist.com/showTask?id=2995104339",
    "duration": {
      "amount": 30,
      "unit": "minute"
    }
  }
]
```

##### Get a task

```
GET /tasks/{task_id}
```

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `task_id` | string | Yes | The task ID |

**Response:** Single task object (same schema as above).

##### Create a task

```
POST /tasks
```

**Headers:**

| Name | Value | Required |
|---|---|---|
| `Content-Type` | `application/json` | Yes |
| `Authorization` | `Bearer {token}` | Yes |
| `X-Request-Id` | UUID string | No (recommended for idempotency) |

**Request Body:**

```json
{
  "content": "Buy groceries",
  "description": "Milk, eggs, bread",
  "project_id": "2203306141",
  "section_id": null,
  "parent_id": null,
  "order": 1,
  "labels": ["shopping"],
  "priority": 4,
  "due_string": "tomorrow at 2pm",
  "due_date": "2026-02-10",
  "due_datetime": "2026-02-10T14:00:00Z",
  "due_lang": "en",
  "assignee_id": "2671355",
  "duration": 30,
  "duration_unit": "minute"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `content` | string | Yes | Task content / title (supports Markdown) |
| `description` | string | No | Task description (supports Markdown) |
| `project_id` | string | No | Project to add task to (defaults to Inbox) |
| `section_id` | string | No | Section within the project |
| `parent_id` | string | No | Parent task ID (creates a subtask) |
| `order` | integer | No | Position within the parent/project |
| `labels` | array[string] | No | Label names to apply |
| `priority` | integer | No | Priority from 1 (normal) to 4 (urgent) |
| `due_string` | string | No | Natural language due date (e.g., "tomorrow at 2pm") |
| `due_date` | string | No | Specific due date (YYYY-MM-DD) |
| `due_datetime` | string | No | Specific due date+time (RFC 3339) |
| `due_lang` | string | No | Language for parsing `due_string` |
| `assignee_id` | string | No | User ID to assign task to (shared projects) |
| `duration` | integer | No | Estimated duration amount |
| `duration_unit` | string | No | `minute` or `day` |

**Response:** Created task object.

##### Update a task

```
POST /tasks/{task_id}
```

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `task_id` | string | Yes | The task ID |

**Request Body:** Same fields as create (all optional).

**Response:** Updated task object.

##### Close (complete) a task

```
POST /tasks/{task_id}/close
```

**Description:** Closes (completes) the specified task. For recurring tasks, advances to the next occurrence.

**Response:** `204 No Content`.

##### Reopen a task

```
POST /tasks/{task_id}/reopen
```

**Description:** Reopens a previously completed task.

**Response:** `204 No Content`.

##### Delete a task

```
DELETE /tasks/{task_id}
```

**Response:** `204 No Content`.

#### 2.5.4 Comments

##### Get all comments

```
GET /comments
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `task_id` | string | Conditional | Task to get comments for (required if `project_id` is not set) |
| `project_id` | string | Conditional | Project to get comments for (required if `task_id` is not set) |

**Response:**

```json
[
  {
    "id": "2992679862",
    "task_id": "2995104339",
    "project_id": null,
    "posted_at": "2026-02-09T12:00:00.000000Z",
    "content": "Need to check the list",
    "attachment": {
      "file_name": "list.pdf",
      "file_type": "application/pdf",
      "file_url": "https://...",
      "resource_type": "file"
    }
  }
]
```

##### Create a comment

```
POST /comments
```

**Request Body:**

```json
{
  "task_id": "2995104339",
  "content": "Don't forget the organic milk"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `task_id` | string | Conditional | Task to comment on |
| `project_id` | string | Conditional | Project to comment on |
| `content` | string | Yes | Comment text (Markdown) |
| `attachment` | object | No | File attachment object |

##### Update a comment

```
POST /comments/{comment_id}
```

**Request Body:**

```json
{
  "content": "Updated comment text"
}
```

##### Delete a comment

```
DELETE /comments/{comment_id}
```

**Response:** `204 No Content`.

#### 2.5.5 Labels

##### Get all personal labels

```
GET /labels
```

**Response:**

```json
[
  {
    "id": "2156154810",
    "name": "shopping",
    "color": "charcoal",
    "order": 1,
    "is_favorite": false
  }
]
```

##### Create a label

```
POST /labels
```

**Request Body:**

```json
{
  "name": "urgent",
  "color": "red",
  "order": 1,
  "is_favorite": false
}
```

##### Update a label

```
POST /labels/{label_id}
```

##### Delete a label

```
DELETE /labels/{label_id}
```

**Response:** `204 No Content`.

#### 2.5.6 Shared Labels

##### Get all shared labels

```
GET /labels/shared
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `omit_personal` | boolean | No | If true, excludes labels that are also personal labels |

##### Rename a shared label

```
POST /labels/shared/rename
```

**Request Body:**

```json
{
  "name": "old_name",
  "new_name": "new_name"
}
```

##### Remove a shared label

```
POST /labels/shared/remove
```

**Request Body:**

```json
{
  "name": "label_to_remove"
}
```

### 2.6 Webhooks / Real-time

Todoist supports webhooks for real-time event notifications.

**Webhook Configuration:** Webhooks are configured through the Todoist App Management Console when registering an OAuth application.

**Webhook URL:** `POST` callback to your registered endpoint.

**Supported Events:**

| Event | Description |
|---|---|
| `item:added` | Task created |
| `item:updated` | Task updated |
| `item:deleted` | Task deleted |
| `item:completed` | Task completed |
| `item:uncompleted` | Task reopened |
| `note:added` | Comment added |
| `note:updated` | Comment updated |
| `note:deleted` | Comment deleted |
| `project:added` | Project created |
| `project:updated` | Project updated |
| `project:deleted` | Project deleted |
| `project:archived` | Project archived |
| `project:unarchived` | Project unarchived |

**Webhook Payload:**

```json
{
  "event_name": "item:added",
  "user_id": "2671355",
  "event_data": {
    "id": "2995104339",
    "content": "Buy groceries",
    "project_id": "2203306141",
    "due": {
      "date": "2026-02-10"
    }
  },
  "version": "9",
  "initiator": {
    "id": "2671355",
    "email": "user@example.com"
  }
}
```

**Verification:** Webhooks include an `X-Todoist-Hmac-SHA256` header. Verify by computing HMAC-SHA256 of the raw request body using your client secret.

### 2.7 Error Handling

| HTTP Code | Description | Esmer Action |
|---|---|---|
| 400 | Bad request (invalid parameters) | Validate inputs |
| 401 | Authentication failed | Re-authenticate user |
| 403 | Forbidden (insufficient permissions) | Check OAuth scopes |
| 404 | Resource not found | Show user-friendly "not found" |
| 429 | Rate limit exceeded | Wait per `Retry-After` header, then retry |
| 500 | Server error | Retry with exponential backoff |
| 503 | Service unavailable | Retry with exponential backoff |

### 2.8 Mobile-Specific Notes

- **Natural Language Parsing:** Use the `due_string` field to pass natural language dates directly (e.g., "tomorrow at 2pm", "every Monday"). This is ideal for voice-driven Esmer commands.
- **Idempotency:** Use the `X-Request-Id` header with a UUID to prevent duplicate task creation on network retries.
- **Offline Queue:** Todoist tokens do not expire, so offline operations can be queued and synced whenever connectivity returns.
- **Sync API:** For bulk operations (e.g., syncing all tasks on app launch), use the Sync API (`/sync/v9/sync`) which returns incremental changes since a given `sync_token`.
- **Small Payloads:** Task objects are compact. Full sync responses can be large; use incremental sync tokens.
- **Deep Linking:** `todoist://task?id={task_id}` opens a task in the Todoist mobile app.

---

## 3. Microsoft To Do (Microsoft Graph API)

### 3.1 Service Overview

Microsoft To Do is Microsoft's task management service, deeply integrated with Outlook, Microsoft 365, and Planner. It supports task lists, tasks with due dates, reminders, recurrence, steps (subtask-like), linked resources, and file attachments. Esmer will use the Microsoft Graph API to manage To Do tasks for users in the Microsoft ecosystem, enabling voice-driven task creation that syncs across Outlook, Teams, and the To Do app.

### 3.2 Authentication

**Method:** OAuth 2.0 (Microsoft Identity Platform v2.0)

| Parameter | Value |
|---|---|
| Authorization URL | `https://login.microsoftonline.com/common/oauth2/v2.0/authorize` |
| Token URL | `https://login.microsoftonline.com/common/oauth2/v2.0/token` |
| Grant Type | `authorization_code` |
| Refresh Token Support | Yes |

**Government Cloud Variants:**

| Environment | Authorization URL |
|---|---|
| US Government (GCC) | `https://login.microsoftonline.us/common/oauth2/v2.0/authorize` |
| US Government DOD | `https://login.microsoftonline.us/common/oauth2/v2.0/authorize` |
| China (21Vianet) | `https://login.chinacloudapi.cn/common/oauth2/v2.0/authorize` |

**Required Scopes:**

| Scope | Description | Required |
|---|---|---|
| `Tasks.ReadWrite` | Read and write access to user's tasks | Yes |
| `Tasks.Read` | Read-only access to user's tasks | Optional (read-only mode) |
| `User.Read` | Read user profile | Yes (for sign-in) |
| `offline_access` | Obtain refresh tokens | Yes |

**Recommendations for Esmer:**
- Register an application in the Microsoft Azure Portal (App Registrations).
- Set Supported account types to "Accounts in any organizational directory and personal Microsoft accounts" for broadest compatibility.
- Request `Tasks.ReadWrite`, `User.Read`, and `offline_access` scopes.
- Access tokens expire in ~3600 seconds; use refresh tokens to renew silently.
- Enterprise users may require admin consent. Handle the `AADSTS65001` (admin consent required) error gracefully.

### 3.3 Base URL

```
https://graph.microsoft.com/v1.0
```

Government cloud:
```
https://graph.microsoft.us/v1.0          (US Gov)
https://microsoftgraph.chinacloudapi.cn/v1.0  (China)
```

### 3.4 Rate Limits

| Limit | Value |
|---|---|
| Per-app per-tenant | 10,000 requests per 10 minutes |
| Per-user per-app | 10,000 requests per 10 minutes |
| Throttle response | HTTP 429 with `Retry-After` header |

Microsoft Graph uses a token-bucket algorithm. When throttled, respect the `Retry-After` header (in seconds).

### 3.5 API Endpoints

#### 3.5.1 Task Lists

##### List all task lists

```
GET /me/todo/lists
```

**Description:** Get all the user's task lists.

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `$top` | integer | No | Number of results to return |
| `$skip` | integer | No | Number of results to skip |
| `$filter` | string | No | OData filter expression |
| `$orderby` | string | No | OData order expression |
| `$select` | string | No | Comma-separated list of properties to return |

**Response:**

```json
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users('user-id')/todo/lists",
  "value": [
    {
      "id": "AAMkADIyAAAhrbPWAAA=",
      "displayName": "Tasks",
      "isOwner": true,
      "isShared": false,
      "wellknownListName": "defaultList"
    }
  ]
}
```

##### Get a task list

```
GET /me/todo/lists/{todoTaskListId}
```

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `todoTaskListId` | string | Yes | The task list ID |

##### Create a task list

```
POST /me/todo/lists
```

**Request Body:**

```json
{
  "displayName": "Work Tasks"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `displayName` | string | Yes | Name of the task list |

**Response:** Created task list object.

##### Update a task list

```
PATCH /me/todo/lists/{todoTaskListId}
```

**Request Body:**

```json
{
  "displayName": "Updated List Name"
}
```

##### Delete a task list

```
DELETE /me/todo/lists/{todoTaskListId}
```

**Response:** `204 No Content`.

#### 3.5.2 Tasks

##### List all tasks in a list

```
GET /me/todo/lists/{todoTaskListId}/tasks
```

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `todoTaskListId` | string | Yes | The task list ID |

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `$top` | integer | No | Page size |
| `$skip` | integer | No | Items to skip |
| `$filter` | string | No | OData filter (e.g., `status eq 'notStarted'`) |
| `$orderby` | string | No | Sort order |
| `$select` | string | No | Fields to include |

**Response:**

```json
{
  "value": [
    {
      "id": "AAMkADIyAAAhrbPXAAA=",
      "title": "Buy groceries",
      "body": {
        "content": "Milk, eggs, bread",
        "contentType": "text"
      },
      "importance": "normal",
      "status": "notStarted",
      "isReminderOn": true,
      "reminderDateTime": {
        "dateTime": "2026-02-10T09:00:00.0000000",
        "timeZone": "UTC"
      },
      "dueDateTime": {
        "dateTime": "2026-02-10T00:00:00.0000000",
        "timeZone": "UTC"
      },
      "completedDateTime": null,
      "createdDateTime": "2026-02-09T12:00:00.0000000Z",
      "lastModifiedDateTime": "2026-02-09T12:00:00.0000000Z",
      "categories": [],
      "hasAttachments": false,
      "recurrence": null
    }
  ]
}
```

##### Get a task

```
GET /me/todo/lists/{todoTaskListId}/tasks/{todoTaskId}
```

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `todoTaskListId` | string | Yes | The task list ID |
| `todoTaskId` | string | Yes | The task ID |

##### Create a task

```
POST /me/todo/lists/{todoTaskListId}/tasks
```

**Request Body:**

```json
{
  "title": "Buy groceries",
  "body": {
    "content": "Milk, eggs, bread",
    "contentType": "text"
  },
  "importance": "high",
  "status": "notStarted",
  "dueDateTime": {
    "dateTime": "2026-02-10T00:00:00",
    "timeZone": "Eastern Standard Time"
  },
  "isReminderOn": true,
  "reminderDateTime": {
    "dateTime": "2026-02-10T09:00:00",
    "timeZone": "Eastern Standard Time"
  },
  "categories": ["Personal"],
  "recurrence": {
    "pattern": {
      "type": "weekly",
      "interval": 1,
      "daysOfWeek": ["monday", "wednesday", "friday"]
    },
    "range": {
      "type": "noEnd",
      "startDate": "2026-02-10"
    }
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `title` | string | Yes | Task title |
| `body` | object | No | Task body with `content` (string) and `contentType` (`text` or `html`) |
| `importance` | string | No | `low`, `normal`, or `high` |
| `status` | string | No | `notStarted`, `inProgress`, `completed`, `waitingOnOthers`, `deferred` |
| `dueDateTime` | object | No | Due date with `dateTime` and `timeZone` |
| `isReminderOn` | boolean | No | Whether to enable reminder |
| `reminderDateTime` | object | No | Reminder date/time with `dateTime` and `timeZone` |
| `categories` | array[string] | No | Outlook categories to apply |
| `recurrence` | object | No | Recurrence pattern and range |

**Response:** Created task object.

##### Update a task

```
PATCH /me/todo/lists/{todoTaskListId}/tasks/{todoTaskId}
```

**Request Body:** Any task fields (all optional for PATCH).

```json
{
  "status": "completed",
  "completedDateTime": {
    "dateTime": "2026-02-09T15:00:00",
    "timeZone": "UTC"
  }
}
```

##### Delete a task

```
DELETE /me/todo/lists/{todoTaskListId}/tasks/{todoTaskId}
```

**Response:** `204 No Content`.

#### 3.5.3 Checklist Items (Steps)

##### List checklist items

```
GET /me/todo/lists/{todoTaskListId}/tasks/{todoTaskId}/checklistItems
```

**Response:**

```json
{
  "value": [
    {
      "id": "e2f9be3c-1c7a-4b5f-9e0e-1a2b3c4d5e6f",
      "displayName": "Buy milk",
      "isChecked": false,
      "createdDateTime": "2026-02-09T12:00:00.0000000Z"
    }
  ]
}
```

##### Create a checklist item

```
POST /me/todo/lists/{todoTaskListId}/tasks/{todoTaskId}/checklistItems
```

**Request Body:**

```json
{
  "displayName": "Buy milk"
}
```

##### Update a checklist item

```
PATCH /me/todo/lists/{todoTaskListId}/tasks/{todoTaskId}/checklistItems/{checklistItemId}
```

**Request Body:**

```json
{
  "displayName": "Buy organic milk",
  "isChecked": true
}
```

##### Delete a checklist item

```
DELETE /me/todo/lists/{todoTaskListId}/tasks/{todoTaskId}/checklistItems/{checklistItemId}
```

#### 3.5.4 Linked Resources

##### List linked resources

```
GET /me/todo/lists/{todoTaskListId}/tasks/{todoTaskId}/linkedResources
```

**Response:**

```json
{
  "value": [
    {
      "id": "e2f9be3c-...",
      "webUrl": "https://example.com/document",
      "applicationName": "Esmer",
      "displayName": "Related Document",
      "externalId": "ext-123"
    }
  ]
}
```

##### Create a linked resource

```
POST /me/todo/lists/{todoTaskListId}/tasks/{todoTaskId}/linkedResources
```

**Request Body:**

```json
{
  "webUrl": "https://example.com/document",
  "applicationName": "Esmer",
  "displayName": "Related Document",
  "externalId": "ext-123"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `webUrl` | string | Yes | URL of the linked resource |
| `applicationName` | string | Yes | Name of the app creating the link |
| `displayName` | string | Yes | Display text for the link |
| `externalId` | string | Yes | External identifier for the resource |

##### Update a linked resource

```
PATCH /me/todo/lists/{todoTaskListId}/tasks/{todoTaskId}/linkedResources/{linkedResourceId}
```

##### Delete a linked resource

```
DELETE /me/todo/lists/{todoTaskListId}/tasks/{todoTaskId}/linkedResources/{linkedResourceId}
```

### 3.6 Webhooks / Real-time

Microsoft Graph supports change notifications (subscriptions) for To Do resources.

**Create a subscription:**

```
POST /subscriptions
```

**Request Body:**

```json
{
  "changeType": "created,updated,deleted",
  "notificationUrl": "https://esmer.example.com/webhooks/microsoft-todo",
  "resource": "/me/todo/lists/{todoTaskListId}/tasks",
  "expirationDateTime": "2026-02-10T12:00:00.000Z",
  "clientState": "secretClientState"
}
```

**Subscription Limits:**
- Maximum expiration: 4230 minutes (roughly 2.94 days) for user-level resources.
- Must renew subscriptions before they expire.
- Microsoft sends a validation request to your `notificationUrl` on creation.

**Notification Payload:**

```json
{
  "value": [
    {
      "subscriptionId": "sub-id",
      "changeType": "created",
      "resource": "me/todo/lists/list-id/tasks/task-id",
      "clientState": "secretClientState",
      "tenantId": "tenant-id"
    }
  ]
}
```

### 3.7 Error Handling

| HTTP Code | Error Code | Description | Esmer Action |
|---|---|---|---|
| 400 | `BadRequest` | Invalid request | Validate inputs |
| 401 | `Unauthorized` | Invalid or expired token | Refresh token; re-auth if refresh fails |
| 403 | `Forbidden` | Insufficient permissions or admin consent required | Check scopes; guide user to request admin consent |
| 404 | `NotFound` | Resource not found | Show "not found" message |
| 409 | `Conflict` | Conflict (e.g., concurrent modification) | Re-fetch and retry |
| 429 | `TooManyRequests` | Rate limit exceeded | Wait per `Retry-After` header |
| 500 | `InternalServerError` | Microsoft server error | Retry with exponential backoff |
| 503 | `ServiceUnavailable` | Service temporarily unavailable | Retry with exponential backoff |

### 3.8 Mobile-Specific Notes

- **Cross-Platform Sync:** Tasks created via Graph API appear instantly in the Microsoft To Do app, Outlook tasks, and Teams.
- **Recurrence Support:** Full recurrence pattern support (daily, weekly, monthly, yearly with complex patterns). Useful for Esmer to create recurring delegated tasks.
- **Rich Body Content:** Task body supports both plain text and HTML. For mobile, prefer `text` content type for performance.
- **Token Management:** Microsoft access tokens expire in ~3600 seconds. Implement silent token refresh using the refresh token. Store tokens in secure device storage.
- **Admin Consent:** Enterprise users may encounter `AADSTS65001`. Esmer should detect this and display a message guiding the user to contact their IT admin.
- **Deep Linking:** `ms-todo://tasks/{taskId}` may open the To Do app on mobile (support varies by platform).
- **Pagination:** Microsoft Graph uses `@odata.nextLink` for pagination. Follow these links to fetch additional pages.

---

## 4. Notion (Notion API)

### 4.1 Service Overview

Notion is an all-in-one workspace combining notes, databases, wikis, and project management. The Notion API allows creating and querying database entries (pages), managing blocks (content), and searching across workspaces. Esmer will use Notion for users who manage tasks and projects as database entries in Notion, enabling delegated task creation, status updates, and content appending from mobile.

### 4.2 Authentication

**Methods:** OAuth 2.0 (public integrations), Internal Integration Token (internal integrations)

| Parameter | Value |
|---|---|
| Authorization URL | `https://api.notion.com/v1/oauth/authorize` |
| Token URL | `https://api.notion.com/v1/oauth/token` |
| Grant Type | `authorization_code` |
| Refresh Token Support | No (access tokens do not expire) |

**OAuth 2.0 (Public Integrations):**

| Field | Description |
|---|---|
| `client_id` | From the integration's Secrets tab |
| `client_secret` | From the integration's Secrets tab |
| `redirect_uri` | Must match the configured redirect URI |
| `response_type` | `code` |

Token exchange uses Basic Auth: `Authorization: Basic base64(client_id:client_secret)`

**Internal Integration Token:**
- Generated from the Notion Integrations dashboard.
- Passed via `Authorization: Bearer {integration_secret}` header.
- The integration must be explicitly shared with each page/database it needs to access.

**Required Capabilities:**

| Capability | Description | Required |
|---|---|---|
| Read content | Read pages, databases, blocks | Yes |
| Update content | Update pages, databases, blocks | Yes |
| Insert content | Create pages, append blocks | Yes |
| Read comments | Read comments on pages and blocks | Optional |
| Create comments | Post comments | Optional |
| User information without email | Access user names and avatars | Recommended |
| User information with email | Access user email addresses | Optional |

**Recommendations for Esmer:**
- Use OAuth 2.0 for multi-user support. Notion tokens do not expire, simplifying token management.
- Users must explicitly share pages/databases with the Esmer integration during the OAuth flow (Notion's consent screen lets users pick which pages to share).
- For maximum functionality, request Read, Update, and Insert content capabilities.
- Always include the `Notion-Version` header in every request.

### 4.3 Base URL

```
https://api.notion.com/v1
```

**Required Header (all requests):**

| Header | Value |
|---|---|
| `Notion-Version` | `2022-06-28` (latest stable) |
| `Authorization` | `Bearer {token}` |
| `Content-Type` | `application/json` |

### 4.4 Rate Limits

| Limit | Value |
|---|---|
| Average rate | 3 requests per second per integration |
| Burst | Short bursts above 3 req/sec are tolerated |
| Throttle response | HTTP 429 with `Retry-After` header |

Notion uses a sliding window rate limiter. The `Retry-After` header indicates seconds to wait.

### 4.5 API Endpoints

#### 4.5.1 Databases

##### Retrieve a database

```
GET /databases/{database_id}
```

**Description:** Retrieves a database object with its schema (properties).

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `database_id` | string (UUID) | Yes | The database ID |

**Response:**

```json
{
  "object": "database",
  "id": "d9824bdc-8445-4327-be8b-5b47500af6ce",
  "created_time": "2026-01-01T00:00:00.000Z",
  "last_edited_time": "2026-02-09T12:00:00.000Z",
  "title": [
    {
      "type": "text",
      "text": { "content": "Task Tracker" },
      "plain_text": "Task Tracker"
    }
  ],
  "properties": {
    "Name": {
      "id": "title",
      "name": "Name",
      "type": "title",
      "title": {}
    },
    "Status": {
      "id": "abc123",
      "name": "Status",
      "type": "status",
      "status": {
        "options": [
          { "id": "1", "name": "Not started", "color": "default" },
          { "id": "2", "name": "In progress", "color": "blue" },
          { "id": "3", "name": "Done", "color": "green" }
        ],
        "groups": []
      }
    },
    "Due Date": {
      "id": "def456",
      "name": "Due Date",
      "type": "date",
      "date": {}
    },
    "Assignee": {
      "id": "ghi789",
      "name": "Assignee",
      "type": "people",
      "people": {}
    },
    "Priority": {
      "id": "jkl012",
      "name": "Priority",
      "type": "select",
      "select": {
        "options": [
          { "id": "a", "name": "High", "color": "red" },
          { "id": "b", "name": "Medium", "color": "yellow" },
          { "id": "c", "name": "Low", "color": "green" }
        ]
      }
    },
    "Tags": {
      "id": "mno345",
      "name": "Tags",
      "type": "multi_select",
      "multi_select": {
        "options": [
          { "id": "x", "name": "frontend", "color": "blue" },
          { "id": "y", "name": "backend", "color": "purple" }
        ]
      }
    }
  },
  "url": "https://www.notion.so/d9824bdc84454327be8b5b47500af6ce",
  "archived": false,
  "is_inline": false
}
```

##### Query a database

```
POST /databases/{database_id}/query
```

**Description:** Gets a list of pages (entries) in a database, with optional filtering and sorting.

**Request Body:**

```json
{
  "filter": {
    "and": [
      {
        "property": "Status",
        "status": {
          "does_not_equal": "Done"
        }
      },
      {
        "property": "Assignee",
        "people": {
          "contains": "user-uuid-here"
        }
      }
    ]
  },
  "sorts": [
    {
      "property": "Due Date",
      "direction": "ascending"
    }
  ],
  "page_size": 50,
  "start_cursor": "cursor-string"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `filter` | object | No | Filter object (supports `and`, `or`, property-specific filters) |
| `sorts` | array | No | Sort criteria |
| `page_size` | integer | No | Max results per page (max 100) |
| `start_cursor` | string | No | Pagination cursor from previous response |

**Response:**

```json
{
  "object": "list",
  "results": [
    {
      "object": "page",
      "id": "page-uuid",
      "created_time": "2026-02-09T12:00:00.000Z",
      "last_edited_time": "2026-02-09T12:00:00.000Z",
      "parent": {
        "type": "database_id",
        "database_id": "d9824bdc-..."
      },
      "properties": {
        "Name": {
          "type": "title",
          "title": [
            { "type": "text", "text": { "content": "Buy groceries" }, "plain_text": "Buy groceries" }
          ]
        },
        "Status": {
          "type": "status",
          "status": { "id": "1", "name": "Not started", "color": "default" }
        },
        "Due Date": {
          "type": "date",
          "date": { "start": "2026-02-10", "end": null, "time_zone": null }
        }
      },
      "url": "https://www.notion.so/page-uuid"
    }
  ],
  "has_more": false,
  "next_cursor": null
}
```

##### Search databases and pages

```
POST /search
```

**Request Body:**

```json
{
  "query": "Task Tracker",
  "filter": {
    "value": "database",
    "property": "object"
  },
  "sort": {
    "direction": "descending",
    "timestamp": "last_edited_time"
  },
  "page_size": 10,
  "start_cursor": null
}
```

#### 4.5.2 Pages (Database Entries / Tasks)

##### Create a page (task entry)

```
POST /pages
```

**Description:** Creates a new page. When the parent is a database, this creates a new entry (task).

**Request Body:**

```json
{
  "parent": {
    "database_id": "d9824bdc-8445-4327-be8b-5b47500af6ce"
  },
  "properties": {
    "Name": {
      "title": [
        {
          "text": { "content": "Buy groceries" }
        }
      ]
    },
    "Status": {
      "status": { "name": "Not started" }
    },
    "Due Date": {
      "date": { "start": "2026-02-10" }
    },
    "Priority": {
      "select": { "name": "High" }
    },
    "Tags": {
      "multi_select": [
        { "name": "frontend" },
        { "name": "backend" }
      ]
    },
    "Assignee": {
      "people": [
        { "id": "user-uuid-here" }
      ]
    }
  },
  "children": [
    {
      "object": "block",
      "type": "paragraph",
      "paragraph": {
        "rich_text": [
          {
            "type": "text",
            "text": { "content": "Task details go here." }
          }
        ]
      }
    },
    {
      "object": "block",
      "type": "to_do",
      "to_do": {
        "rich_text": [{ "type": "text", "text": { "content": "Subtask 1" } }],
        "checked": false
      }
    }
  ]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `parent` | object | Yes | Parent database or page |
| `properties` | object | Yes | Property values matching the database schema |
| `children` | array | No | Block content for the page body |
| `icon` | object | No | Page icon (emoji or external URL) |
| `cover` | object | No | Cover image |

**Response:** Created page object.

##### Retrieve a page

```
GET /pages/{page_id}
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `filter_properties` | string | No | Comma-separated property IDs to return |

##### Update a page (update task properties)

```
PATCH /pages/{page_id}
```

**Request Body:**

```json
{
  "properties": {
    "Status": {
      "status": { "name": "Done" }
    }
  },
  "archived": false
}
```

##### Archive a page

```
PATCH /pages/{page_id}
```

**Request Body:**

```json
{
  "archived": true
}
```

#### 4.5.3 Blocks

##### Retrieve block children

```
GET /blocks/{block_id}/children
```

**Description:** Returns the children blocks of a given block or page.

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `page_size` | integer | No | Max results (max 100) |
| `start_cursor` | string | No | Pagination cursor |

**Response:**

```json
{
  "object": "list",
  "results": [
    {
      "object": "block",
      "id": "block-uuid",
      "type": "paragraph",
      "paragraph": {
        "rich_text": [
          { "type": "text", "text": { "content": "Some content" }, "plain_text": "Some content" }
        ]
      },
      "has_children": false
    }
  ],
  "has_more": false
}
```

##### Append block children

```
PATCH /blocks/{block_id}/children
```

**Description:** Appends new blocks as children of the specified block or page.

**Request Body:**

```json
{
  "children": [
    {
      "object": "block",
      "type": "paragraph",
      "paragraph": {
        "rich_text": [
          { "type": "text", "text": { "content": "Added by Esmer" } }
        ]
      }
    },
    {
      "object": "block",
      "type": "to_do",
      "to_do": {
        "rich_text": [{ "type": "text", "text": { "content": "New subtask" } }],
        "checked": false
      }
    }
  ]
}
```

**Supported block types for Esmer:**

| Block Type | Description |
|---|---|
| `paragraph` | Text paragraph |
| `heading_1` | Heading level 1 |
| `heading_2` | Heading level 2 |
| `heading_3` | Heading level 3 |
| `bulleted_list_item` | Bulleted list item |
| `numbered_list_item` | Numbered list item |
| `to_do` | Checkbox / to-do item |
| `toggle` | Toggle block |
| `code` | Code block |
| `callout` | Callout block |
| `divider` | Horizontal divider |
| `bookmark` | URL bookmark |

##### Delete a block

```
DELETE /blocks/{block_id}
```

**Response:** Returns the deleted block with `archived: true`.

#### 4.5.4 Users

##### List all users

```
GET /users
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `page_size` | integer | No | Max results |
| `start_cursor` | string | No | Pagination cursor |

**Response:**

```json
{
  "object": "list",
  "results": [
    {
      "object": "user",
      "id": "user-uuid",
      "type": "person",
      "name": "Jane Doe",
      "avatar_url": "https://...",
      "person": {
        "email": "jane@example.com"
      }
    }
  ]
}
```

##### Retrieve a user

```
GET /users/{user_id}
```

##### Retrieve the bot user

```
GET /users/me
```

**Description:** Retrieves the bot/integration user. Useful for identifying what workspace the integration is connected to.

#### 4.5.5 Comments

##### Retrieve comments

```
GET /comments
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `block_id` | string | Yes | The block or page ID to get comments for |
| `page_size` | integer | No | Max results |
| `start_cursor` | string | No | Pagination cursor |

##### Create a comment

```
POST /comments
```

**Request Body:**

```json
{
  "parent": {
    "page_id": "page-uuid"
  },
  "rich_text": [
    {
      "type": "text",
      "text": { "content": "Task delegated by Esmer" }
    }
  ]
}
```

### 4.6 Webhooks / Real-time

The Notion API does **not** support webhooks. Esmer must use polling.

**Polling Strategy for Esmer:**
- Use the `last_edited_time` filter when querying databases to detect changes since last sync.
- Poll every 30-60 seconds for active users.
- Use the Search endpoint with `last_edited_time` sort to find recently modified pages.
- Notion provides a Trigger node (polling-based) in n8n, confirming there is no native push mechanism.

### 4.7 Error Handling

| HTTP Code | Error Code | Description | Esmer Action |
|---|---|---|---|
| 400 | `validation_error` | Invalid request body | Validate property names/types against database schema |
| 401 | `unauthorized` | Invalid token | Re-authenticate |
| 403 | `restricted_resource` | Integration not shared with this resource | Prompt user to share the page/database with the Esmer integration |
| 404 | `object_not_found` | Page, database, or block not found | Verify ID; may need user to share resource |
| 409 | `conflict_error` | Transaction conflict | Retry after brief delay |
| 429 | `rate_limited` | Rate limit exceeded | Respect `Retry-After` header |
| 500 | `internal_server_error` | Notion server error | Retry with exponential backoff |
| 502 | `bad_gateway` | Notion gateway error | Retry with exponential backoff |
| 503 | `service_unavailable` | Notion unavailable | Retry with exponential backoff |

**Common Issue:** If a page/database is not shared with the integration, all API calls will return `404` or `403`. Esmer must guide the user through sharing their Notion pages with the integration.

### 4.8 Mobile-Specific Notes

- **Dynamic Schema:** Every Notion database has a different schema (properties). Esmer must first fetch the database schema (`GET /databases/{id}`) before creating entries, to know which properties exist and their types.
- **Rich Text Model:** All text in Notion uses a rich text array format. Esmer should use the simple `text` type for most operations.
- **Large Payloads:** Notion responses can be verbose (nested rich text, property definitions). Consider using `filter_properties` to limit returned data on mobile.
- **Page vs Database Page:** Standalone pages and database entries (pages) use the same API. The `parent` field determines the type.
- **Block Limits:** A single request can append up to 100 blocks. For longer content, batch into multiple requests.
- **Offline Strategy:** Cache database schemas and recently accessed pages. Queue create/update operations for offline sync.
- **Deep Linking:** `notion://www.notion.so/{page_id}` opens a page in the Notion mobile app.

---

## 5. Asana (Asana REST API)

### 5.1 Service Overview

Asana is a team project management platform supporting projects, tasks, subtasks, sections, custom fields, tags, comments, and team collaboration. Esmer will use Asana for users who manage team work in Asana, enabling mobile-first task creation, assignment, status updates, and comment posting on behalf of the user.

### 5.2 Authentication

**Methods:** OAuth 2.0, Personal Access Token (PAT)

| Parameter | Value |
|---|---|
| Authorization URL | `https://app.asana.com/-/oauth_authorize` |
| Token URL | `https://app.asana.com/-/oauth_token` |
| Grant Type | `authorization_code` |
| Refresh Token Support | Yes (access tokens expire in 1 hour) |

**Required Scopes:**

Asana OAuth does not use granular scopes. An authorized token has full access to all resources the user can access. Permissions are determined by the user's Asana role and project membership.

| Scope | Description |
|---|---|
| `default` | Full access to all resources accessible by the authenticated user |

**PAT Alternative:**
- Generated from Asana Developer Console > Personal Access Tokens.
- Passed via `Authorization: Bearer {pat}` header.
- Full access equivalent to OAuth token.

**Recommendations for Esmer:**
- Use OAuth 2.0 for multi-user deployments. Access tokens expire in 3600 seconds; implement refresh token logic.
- PATs are suitable for single-user testing but should not be used in production.
- Store the `refresh_token` securely; it does not expire unless revoked.

### 5.3 Base URL

```
https://app.asana.com/api/1.0
```

### 5.4 Rate Limits

| Limit | Value |
|---|---|
| Standard | ~1,500 requests per minute per PAT/OAuth token |
| Concurrency limit | 50 concurrent requests per user/app combination |
| Search endpoint | 60 requests per minute |

Asana returns rate limit headers:
- `X-Asana-Rate-Limit-Remaining`: Requests remaining in current window.
- `Retry-After`: Seconds to wait when throttled (HTTP 429).

### 5.5 API Endpoints

#### 5.5.1 Projects

##### Get all projects

```
GET /projects
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `workspace` | string | Recommended | Workspace GID to filter by |
| `team` | string | No | Team GID to filter by |
| `archived` | boolean | No | Filter by archived status |
| `opt_fields` | string | No | Comma-separated fields to include |
| `limit` | integer | No | Results per page (max 100) |
| `offset` | string | No | Pagination offset token |

**Response:**

```json
{
  "data": [
    {
      "gid": "1203456789",
      "name": "Product Launch",
      "resource_type": "project",
      "archived": false,
      "color": "light-green",
      "created_at": "2026-01-01T00:00:00.000Z",
      "current_status": null,
      "due_date": "2026-03-01",
      "owner": {
        "gid": "9876543210",
        "name": "Jane Doe",
        "resource_type": "user"
      },
      "workspace": {
        "gid": "1111111111",
        "name": "My Workspace",
        "resource_type": "workspace"
      }
    }
  ],
  "next_page": {
    "offset": "eyJ0eXAiOiJKV1...",
    "path": "/projects?offset=eyJ0eXAiOiJKV1...",
    "uri": "https://app.asana.com/api/1.0/projects?offset=eyJ0eXAiOiJKV1..."
  }
}
```

##### Get a project

```
GET /projects/{project_gid}
```

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `project_gid` | string | Yes | The project GID |

##### Create a project

```
POST /projects
```

**Request Body:**

```json
{
  "data": {
    "name": "Product Launch",
    "workspace": "1111111111",
    "team": "2222222222",
    "notes": "Project for Q1 product launch",
    "color": "light-green",
    "default_view": "list",
    "due_date": "2026-03-01",
    "start_on": "2026-01-15",
    "is_template": false,
    "public": true
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Project name |
| `workspace` | string | Yes | Workspace GID |
| `team` | string | No | Team GID (required for team workspaces) |
| `notes` | string | No | Project description |
| `color` | string | No | Color label |
| `default_view` | string | No | `list`, `board`, `calendar`, `timeline` |
| `due_date` | string | No | Due date (YYYY-MM-DD) |
| `start_on` | string | No | Start date |
| `public` | boolean | No | Whether the project is visible to the workspace |

##### Update a project

```
PUT /projects/{project_gid}
```

**Request Body:** Same fields as create (all optional).

##### Delete a project

```
DELETE /projects/{project_gid}
```

**Response:** `{ "data": {} }` with `200 OK`.

#### 5.5.2 Tasks

##### Get all tasks

```
GET /tasks
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `project` | string | Conditional | Project GID (one of project, tag, section, user_task_list, or assignee required) |
| `section` | string | Conditional | Section GID |
| `assignee` | string | Conditional | User GID or `me` |
| `workspace` | string | Conditional | Required when filtering by assignee |
| `completed_since` | string | No | ISO 8601 datetime (include tasks completed after this time) |
| `modified_since` | string | No | ISO 8601 datetime |
| `opt_fields` | string | No | Comma-separated fields to include |
| `limit` | integer | No | Max 100 |
| `offset` | string | No | Pagination token |

**Response:**

```json
{
  "data": [
    {
      "gid": "3333333333",
      "name": "Buy groceries",
      "resource_type": "task",
      "assignee": {
        "gid": "9876543210",
        "name": "Jane Doe"
      },
      "completed": false,
      "completed_at": null,
      "created_at": "2026-02-09T12:00:00.000Z",
      "due_on": "2026-02-10",
      "due_at": "2026-02-10T14:00:00.000Z",
      "notes": "Milk, eggs, bread",
      "html_notes": "<body>Milk, eggs, bread</body>",
      "projects": [
        { "gid": "1203456789", "name": "Product Launch" }
      ],
      "tags": [
        { "gid": "4444444444", "name": "urgent" }
      ],
      "custom_fields": [
        {
          "gid": "5555555555",
          "name": "Priority",
          "display_value": "High",
          "type": "enum",
          "enum_value": { "gid": "6666666666", "name": "High", "color": "red" }
        }
      ],
      "parent": null,
      "num_subtasks": 2,
      "memberships": [
        {
          "project": { "gid": "1203456789", "name": "Product Launch" },
          "section": { "gid": "7777777777", "name": "To Do" }
        }
      ],
      "permalink_url": "https://app.asana.com/0/1203456789/3333333333"
    }
  ]
}
```

##### Get a task

```
GET /tasks/{task_gid}
```

##### Create a task

```
POST /tasks
```

**Request Body:**

```json
{
  "data": {
    "name": "Buy groceries",
    "assignee": "me",
    "projects": ["1203456789"],
    "memberships": [
      {
        "project": "1203456789",
        "section": "7777777777"
      }
    ],
    "notes": "Milk, eggs, bread",
    "html_notes": "<body><strong>Milk</strong>, eggs, bread</body>",
    "due_on": "2026-02-10",
    "due_at": "2026-02-10T14:00:00.000Z",
    "start_on": "2026-02-09",
    "completed": false,
    "custom_fields": {
      "5555555555": "6666666666"
    },
    "tags": ["4444444444"],
    "parent": null,
    "resource_subtype": "default_task"
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Task name |
| `assignee` | string | No | User GID or `me` |
| `projects` | array[string] | No | Project GIDs to add task to |
| `memberships` | array[object] | No | Project + section pairs |
| `notes` | string | No | Plain text description |
| `html_notes` | string | No | HTML description |
| `due_on` | string | No | Due date (YYYY-MM-DD) |
| `due_at` | string | No | Due date+time (ISO 8601) |
| `start_on` | string | No | Start date |
| `completed` | boolean | No | Whether the task is completed |
| `custom_fields` | object | No | Map of custom field GID to value |
| `tags` | array[string] | No | Tag GIDs |
| `parent` | string | No | Parent task GID (for subtasks) |

##### Update a task

```
PUT /tasks/{task_gid}
```

**Request Body:** Same fields as create (all optional).

##### Delete a task

```
DELETE /tasks/{task_gid}
```

##### Search tasks

```
GET /workspaces/{workspace_gid}/tasks/search
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `text` | string | No | Text to search for |
| `assignee.any` | string | No | Comma-separated user GIDs |
| `projects.any` | string | No | Comma-separated project GIDs |
| `tags.any` | string | No | Comma-separated tag GIDs |
| `completed` | boolean | No | Filter by completion |
| `is_subtask` | boolean | No | Filter subtasks |
| `due_on.before` | string | No | Due date filter (YYYY-MM-DD) |
| `due_on.after` | string | No | Due date filter |
| `modified_at.after` | string | No | Modified since filter |
| `sort_by` | string | No | `due_date`, `created_at`, `completed_at`, `modified_at`, `likes` |
| `sort_ascending` | boolean | No | Sort direction |

**Note:** Search endpoint has a stricter rate limit of 60 requests per minute.

##### Add a task to a project

```
POST /tasks/{task_gid}/addProject
```

**Request Body:**

```json
{
  "data": {
    "project": "1203456789",
    "section": "7777777777"
  }
}
```

##### Remove a task from a project

```
POST /tasks/{task_gid}/removeProject
```

**Request Body:**

```json
{
  "data": {
    "project": "1203456789"
  }
}
```

#### 5.5.3 Subtasks

##### Get subtasks

```
GET /tasks/{task_gid}/subtasks
```

##### Create a subtask

```
POST /tasks/{task_gid}/subtasks
```

**Request Body:** Same schema as creating a task (without the `parent` field since it is implied).

#### 5.5.4 Task Comments (Stories)

##### Get comments on a task

```
GET /tasks/{task_gid}/stories
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `opt_fields` | string | No | Fields to include |
| `limit` | integer | No | Max results |
| `offset` | string | No | Pagination token |

**Response:**

```json
{
  "data": [
    {
      "gid": "8888888888",
      "resource_type": "story",
      "type": "comment",
      "text": "Great progress on this task!",
      "html_text": "<body>Great progress on this task!</body>",
      "created_at": "2026-02-09T12:00:00.000Z",
      "created_by": {
        "gid": "9876543210",
        "name": "Jane Doe"
      }
    }
  ]
}
```

##### Add a comment to a task

```
POST /tasks/{task_gid}/stories
```

**Request Body:**

```json
{
  "data": {
    "text": "Delegated by Esmer on behalf of user",
    "html_text": "<body><strong>Delegated</strong> by Esmer on behalf of user</body>"
  }
}
```

##### Delete a comment

```
DELETE /stories/{story_gid}
```

#### 5.5.5 Task Tags

##### Add a tag to a task

```
POST /tasks/{task_gid}/addTag
```

**Request Body:**

```json
{
  "data": {
    "tag": "4444444444"
  }
}
```

##### Remove a tag from a task

```
POST /tasks/{task_gid}/removeTag
```

**Request Body:**

```json
{
  "data": {
    "tag": "4444444444"
  }
}
```

#### 5.5.6 Users

##### Get current user

```
GET /users/me
```

##### Get a user

```
GET /users/{user_gid}
```

##### Get all users in a workspace

```
GET /workspaces/{workspace_gid}/users
```

### 5.6 Webhooks / Real-time

Asana supports webhooks for real-time notifications.

**Create a webhook:**

```
POST /webhooks
```

**Request Body:**

```json
{
  "data": {
    "resource": "1203456789",
    "target": "https://esmer.example.com/webhooks/asana",
    "filters": [
      {
        "resource_type": "task",
        "action": "changed",
        "fields": ["completed", "assignee", "due_on"]
      }
    ]
  }
}
```

**Handshake:** Asana sends an initial request with `X-Hook-Secret` header. You must respond with `200` and echo the header back.

**Webhook Events:**

| Event | Description |
|---|---|
| `added` | Resource created |
| `changed` | Resource modified |
| `deleted` | Resource deleted |
| `removed` | Resource removed from parent |
| `undeleted` | Resource restored |

**Webhook Payload:**

```json
{
  "events": [
    {
      "action": "changed",
      "created_at": "2026-02-09T12:00:00.000Z",
      "parent": { "gid": "1203456789", "resource_type": "project" },
      "resource": { "gid": "3333333333", "resource_type": "task" },
      "user": { "gid": "9876543210", "resource_type": "user" },
      "change": {
        "field": "completed",
        "action": "changed",
        "new_value": true
      }
    }
  ]
}
```

**Verification:** Validate using HMAC-SHA256 of the request body with the `X-Hook-Secret` from the handshake.

### 5.7 Error Handling

| HTTP Code | Description | Esmer Action |
|---|---|---|
| 400 | Invalid request | Validate parameters and field types |
| 401 | Not authorized | Refresh token; re-authenticate |
| 402 | Payment required (premium feature) | Inform user feature requires paid plan |
| 403 | Forbidden | Check user permissions for the resource |
| 404 | Not found | Verify GID; resource may have been deleted |
| 429 | Rate limited | Wait per `Retry-After` header |
| 500 | Server error | Retry with exponential backoff |

Asana error responses include detailed error messages:

```json
{
  "errors": [
    {
      "message": "project: Missing input",
      "help": "For more information...",
      "phrase": "6 sad hamsters"
    }
  ]
}
```

### 5.8 Mobile-Specific Notes

- **opt_fields:** Always use `opt_fields` to request only the fields Esmer needs. This significantly reduces response size for mobile.
- **Compact Responses:** By default, Asana returns compact representations (GID + name only). Use `opt_fields` to expand.
- **Pagination:** Use `limit` + `offset` pagination. Default page size is 20; max is 100.
- **User Task List:** Use `GET /users/me/user_task_list?workspace={wid}` then query that list to get the user's "My Tasks" view.
- **Deep Linking:** `https://app.asana.com/0/{project_gid}/{task_gid}` opens a task in both web and mobile (Asana app handles universal links).
- **Token Refresh:** Access tokens expire in 1 hour. Implement proactive refresh (refresh when 80% of TTL has elapsed).
- **Offline Queue:** Queue task creation and updates locally. Asana does not provide a bulk create endpoint; process queue items sequentially.

---

## 6. ClickUp (ClickUp API v2)

### 6.1 Service Overview

ClickUp is a comprehensive project management platform with a deep hierarchy: Workspace > Space > Folder > List > Task. It supports tasks, subtasks, checklists, comments, custom fields, time tracking, goals, tags, dependencies, and multiple views. Esmer will use ClickUp for users managing complex project workflows, enabling task creation, status changes, comment posting, and time entry management from mobile.

### 6.2 Authentication

**Methods:** OAuth 2.0, Personal API Token

| Parameter | Value |
|---|---|
| Authorization URL | `https://app.clickup.com/api` (OAuth redirect) |
| Token URL | `https://app.clickup.com/api/v2/oauth/token` |
| Grant Type | `authorization_code` |
| Refresh Token Support | No (ClickUp OAuth tokens do not expire unless revoked) |

**Personal API Token:**
- Found in ClickUp Settings > Apps > API Token.
- Passed via `Authorization: {token}` header (no "Bearer" prefix for personal tokens).

**OAuth 2.0 Flow:**

1. Redirect user to: `https://app.clickup.com/api?client_id={client_id}&redirect_uri={redirect_uri}`
2. User authorizes; ClickUp redirects with `?code={code}`
3. Exchange code: `POST https://app.clickup.com/api/v2/oauth/token` with `client_id`, `client_secret`, `code`

**Recommendations for Esmer:**
- Use OAuth 2.0 for multi-user support. Tokens do not expire, simplifying management.
- Personal API tokens use `Authorization: {token}` while OAuth tokens use `Authorization: Bearer {token}`.
- ClickUp does not use scopes; the token inherits the user's workspace permissions.

### 6.3 Base URL

```
https://api.clickup.com/api/v2
```

### 6.4 Rate Limits

| Limit | Value |
|---|---|
| Per token | 100 requests per minute |
| Per token (sustained) | 10 requests per second burst, 100/min average |
| Throttle response | HTTP 429 |

When rate limited, ClickUp returns `429 Too Many Requests`. Implement exponential backoff. ClickUp does not provide a `Retry-After` header consistently; wait at least 60 seconds before retrying.

### 6.5 API Endpoints

#### 6.5.1 Workspaces (Teams)

##### Get authorized teams/workspaces

```
GET /team
```

**Response:**

```json
{
  "teams": [
    {
      "id": "1234567",
      "name": "My Workspace",
      "color": "#000000",
      "avatar": "https://...",
      "members": [
        {
          "user": {
            "id": 123,
            "username": "janedoe",
            "email": "jane@example.com",
            "profilePicture": "https://..."
          }
        }
      ]
    }
  ]
}
```

#### 6.5.2 Spaces

##### Get spaces

```
GET /team/{team_id}/space
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `archived` | boolean | No | Include archived spaces |

**Response:**

```json
{
  "spaces": [
    {
      "id": "2345678",
      "name": "Engineering",
      "private": false,
      "statuses": [
        { "id": "s1", "status": "to do", "type": "open", "orderindex": 0, "color": "#d3d3d3" },
        { "id": "s2", "status": "in progress", "type": "custom", "orderindex": 1, "color": "#4194f6" },
        { "id": "s3", "status": "done", "type": "closed", "orderindex": 2, "color": "#6bc950" }
      ],
      "multiple_assignees": true,
      "features": {
        "due_dates": { "enabled": true },
        "time_tracking": { "enabled": true },
        "tags": { "enabled": true },
        "checklists": { "enabled": true },
        "custom_fields": { "enabled": true }
      }
    }
  ]
}
```

#### 6.5.3 Folders

##### Get folders

```
GET /space/{space_id}/folder
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `archived` | boolean | No | Include archived folders |

##### Create a folder

```
POST /space/{space_id}/folder
```

**Request Body:**

```json
{
  "name": "Sprint 1"
}
```

##### Update a folder

```
PUT /folder/{folder_id}
```

##### Delete a folder

```
DELETE /folder/{folder_id}
```

#### 6.5.4 Lists

##### Get lists

```
GET /folder/{folder_id}/list
```

##### Get folderless lists

```
GET /space/{space_id}/list
```

##### Create a list

```
POST /folder/{folder_id}/list
```

**Request Body:**

```json
{
  "name": "Backlog",
  "content": "All backlog items",
  "due_date": 1709251200000,
  "due_date_time": false,
  "priority": 1,
  "assignee": 123,
  "status": "red"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | List name |
| `content` | string | No | Description |
| `due_date` | integer | No | Due date as Unix timestamp in ms |
| `due_date_time` | boolean | No | Whether due_date includes a time |
| `priority` | integer | No | Priority (1=urgent, 2=high, 3=normal, 4=low) |
| `assignee` | integer | No | User ID for default assignee |
| `status` | string | No | Color status indicator |

##### Get list custom fields

```
GET /list/{list_id}/field
```

**Response:**

```json
{
  "fields": [
    {
      "id": "cf_123",
      "name": "Story Points",
      "type": "number",
      "type_config": {},
      "required": false
    },
    {
      "id": "cf_456",
      "name": "Sprint",
      "type": "drop_down",
      "type_config": {
        "options": [
          { "id": "opt1", "name": "Sprint 1", "color": "#000000", "orderindex": 0 }
        ]
      },
      "required": false
    }
  ]
}
```

##### Get list members

```
GET /list/{list_id}/member
```

##### Update a list

```
PUT /list/{list_id}
```

##### Delete a list

```
DELETE /list/{list_id}
```

#### 6.5.5 Tasks

##### Get tasks

```
GET /list/{list_id}/task
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `archived` | boolean | No | Include archived tasks |
| `page` | integer | No | Page number (0-indexed) |
| `order_by` | string | No | `id`, `created`, `updated`, `due_date` |
| `reverse` | boolean | No | Reverse sort order |
| `subtasks` | boolean | No | Include subtasks |
| `statuses[]` | string | No | Filter by status (repeatable) |
| `include_closed` | boolean | No | Include closed tasks |
| `assignees[]` | string | No | Filter by assignee IDs |
| `tags[]` | string | No | Filter by tag names |
| `due_date_gt` | integer | No | Due date after (Unix ms) |
| `due_date_lt` | integer | No | Due date before (Unix ms) |
| `date_created_gt` | integer | No | Created after (Unix ms) |
| `date_created_lt` | integer | No | Created before (Unix ms) |
| `date_updated_gt` | integer | No | Updated after (Unix ms) |
| `date_updated_lt` | integer | No | Updated before (Unix ms) |
| `custom_fields` | string | No | JSON array of custom field filters |

**Response:**

```json
{
  "tasks": [
    {
      "id": "abc123",
      "custom_id": null,
      "name": "Buy groceries",
      "text_content": "Milk, eggs, bread",
      "description": "Milk, eggs, bread",
      "status": {
        "id": "s1",
        "status": "to do",
        "color": "#d3d3d3",
        "type": "open",
        "orderindex": 0
      },
      "orderindex": "1.00000000000000000000",
      "date_created": "1707465600000",
      "date_updated": "1707465600000",
      "date_closed": null,
      "date_done": null,
      "creator": {
        "id": 123,
        "username": "janedoe",
        "email": "jane@example.com"
      },
      "assignees": [
        {
          "id": 123,
          "username": "janedoe",
          "email": "jane@example.com",
          "profilePicture": "https://..."
        }
      ],
      "watchers": [],
      "checklists": [],
      "tags": [
        { "name": "urgent", "tag_fg": "#FFFFFF", "tag_bg": "#FF0000" }
      ],
      "parent": null,
      "priority": {
        "id": "2",
        "priority": "high",
        "color": "#ffcc00",
        "orderindex": "2"
      },
      "due_date": "1707552000000",
      "start_date": null,
      "points": null,
      "time_estimate": null,
      "time_spent": 0,
      "custom_fields": [
        {
          "id": "cf_123",
          "name": "Story Points",
          "type": "number",
          "value": 5
        }
      ],
      "dependencies": [],
      "linked_tasks": [],
      "team_id": "1234567",
      "url": "https://app.clickup.com/t/abc123",
      "list": { "id": "list123", "name": "Backlog" },
      "folder": { "id": "folder123", "name": "Sprint 1" },
      "space": { "id": "2345678" }
    }
  ]
}
```

##### Get a task

```
GET /task/{task_id}
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `custom_task_ids` | boolean | No | If true, `task_id` is treated as a custom task ID |
| `team_id` | string | No | Required when `custom_task_ids` is true |
| `include_subtasks` | boolean | No | Include subtasks in response |
| `include_markdown_description` | boolean | No | Include markdown description field |

##### Create a task

```
POST /list/{list_id}/task
```

**Request Body:**

```json
{
  "name": "Buy groceries",
  "description": "Milk, eggs, bread",
  "markdown_description": "**Milk**, eggs, bread",
  "assignees": [123],
  "tags": ["urgent"],
  "status": "to do",
  "priority": 2,
  "due_date": 1707552000000,
  "due_date_time": true,
  "time_estimate": 3600000,
  "start_date": 1707465600000,
  "start_date_time": false,
  "notify_all": false,
  "parent": null,
  "links_to": null,
  "check_required_custom_fields": true,
  "custom_fields": [
    {
      "id": "cf_123",
      "value": 5
    }
  ]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Task name |
| `description` | string | No | Plain text description |
| `markdown_description` | string | No | Markdown description |
| `assignees` | array[integer] | No | User IDs to assign |
| `tags` | array[string] | No | Tag names |
| `status` | string | No | Status name |
| `priority` | integer | No | 1=urgent, 2=high, 3=normal, 4=low, null=none |
| `due_date` | integer | No | Unix timestamp in ms |
| `due_date_time` | boolean | No | Whether due_date includes time |
| `time_estimate` | integer | No | Time estimate in ms |
| `start_date` | integer | No | Start date as Unix ms |
| `start_date_time` | boolean | No | Whether start_date includes time |
| `parent` | string | No | Parent task ID (creates subtask) |
| `custom_fields` | array[object] | No | Custom field values |
| `notify_all` | boolean | No | Notify assignees and watchers |

##### Update a task

```
PUT /task/{task_id}
```

**Request Body:** Same fields as create (all optional).

##### Delete a task

```
DELETE /task/{task_id}
```

##### Set a custom field value

```
POST /task/{task_id}/field/{field_id}
```

**Request Body:**

```json
{
  "value": 8
}
```

Value format depends on field type:
- `number`: numeric value
- `drop_down`: option order index or option ID
- `labels`: array of label IDs
- `date`: Unix timestamp in ms
- `text`: string
- `checkbox`: boolean

##### Get task members

```
GET /task/{task_id}/member
```

#### 6.5.6 Task Lists (Multi-list membership)

##### Add a task to a list

```
POST /list/{list_id}/task/{task_id}
```

##### Remove a task from a list

```
DELETE /list/{list_id}/task/{task_id}
```

#### 6.5.7 Task Tags

##### Add tag to a task

```
POST /task/{task_id}/tag/{tag_name}
```

##### Remove tag from a task

```
DELETE /task/{task_id}/tag/{tag_name}
```

#### 6.5.8 Task Dependencies

##### Add a dependency

```
POST /task/{task_id}/dependency
```

**Request Body:**

```json
{
  "depends_on": "other_task_id",
  "dependency_of": null
}
```

##### Delete a dependency

```
DELETE /task/{task_id}/dependency
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `depends_on` | string | Conditional | Task ID this depends on |
| `dependency_of` | string | Conditional | Task ID that depends on this |

#### 6.5.9 Checklists

##### Create a checklist

```
POST /task/{task_id}/checklist
```

**Request Body:**

```json
{
  "name": "Shopping List"
}
```

##### Update a checklist

```
PUT /checklist/{checklist_id}
```

##### Delete a checklist

```
DELETE /checklist/{checklist_id}
```

##### Create a checklist item

```
POST /checklist/{checklist_id}/checklist_item
```

**Request Body:**

```json
{
  "name": "Milk",
  "assignee": 123
}
```

##### Update a checklist item

```
PUT /checklist/{checklist_id}/checklist_item/{checklist_item_id}
```

**Request Body:**

```json
{
  "name": "Organic Milk",
  "resolved": true,
  "assignee": 123,
  "parent": null
}
```

##### Delete a checklist item

```
DELETE /checklist/{checklist_id}/checklist_item/{checklist_item_id}
```

#### 6.5.10 Comments

##### Get task comments

```
GET /task/{task_id}/comment
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `start` | integer | No | Start offset for pagination |
| `start_id` | string | No | Comment ID to start from |

**Response:**

```json
{
  "comments": [
    {
      "id": "comm_123",
      "comment": [
        { "text": "Working on this now" }
      ],
      "comment_text": "Working on this now",
      "user": {
        "id": 123,
        "username": "janedoe",
        "email": "jane@example.com"
      },
      "date": "1707465600000"
    }
  ]
}
```

##### Create a task comment

```
POST /task/{task_id}/comment
```

**Request Body:**

```json
{
  "comment_text": "Delegated by Esmer",
  "assignee": 123,
  "notify_all": false
}
```

##### Create a list comment

```
POST /list/{list_id}/comment
```

##### Update a comment

```
PUT /comment/{comment_id}
```

##### Delete a comment

```
DELETE /comment/{comment_id}
```

#### 6.5.11 Goals

##### Get goals

```
GET /team/{team_id}/goal
```

##### Create a goal

```
POST /team/{team_id}/goal
```

**Request Body:**

```json
{
  "name": "Q1 OKR: Ship v2.0",
  "due_date": 1711929600000,
  "description": "Ship version 2.0 by end of Q1",
  "multiple_owners": true,
  "owners": [123],
  "color": "#4194f6"
}
```

##### Update a goal

```
PUT /goal/{goal_id}
```

##### Delete a goal

```
DELETE /goal/{goal_id}
```

##### Create a key result

```
POST /goal/{goal_id}/key_result
```

**Request Body:**

```json
{
  "name": "Complete 80% of stories",
  "owners": [123],
  "type": "percentage",
  "steps_start": 0,
  "steps_end": 100,
  "unit": "%",
  "task_ids": ["abc123"],
  "list_ids": ["list123"]
}
```

##### Update a key result

```
PUT /key_result/{key_result_id}
```

##### Delete a key result

```
DELETE /key_result/{key_result_id}
```

#### 6.5.12 Space Tags

##### Get space tags

```
GET /space/{space_id}/tag
```

##### Create a space tag

```
POST /space/{space_id}/tag
```

**Request Body:**

```json
{
  "tag": {
    "name": "bug",
    "tag_fg": "#FFFFFF",
    "tag_bg": "#FF0000"
  }
}
```

##### Update a space tag

```
PUT /space/{space_id}/tag/{tag_name}
```

##### Delete a space tag

```
DELETE /space/{space_id}/tag/{tag_name}
```

#### 6.5.13 Time Tracking

##### Get time entries

```
GET /team/{team_id}/time_entries
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `start_date` | integer | No | Unix ms |
| `end_date` | integer | No | Unix ms |
| `assignee` | integer | No | User ID |
| `include_task_tags` | boolean | No | Include tags |
| `include_location_names` | boolean | No | Include space/folder/list names |

##### Create a time entry

```
POST /team/{team_id}/time_entries
```

**Request Body:**

```json
{
  "description": "Working on task",
  "tid": "abc123",
  "start": 1707465600000,
  "duration": 3600000,
  "assignee": 123,
  "billable": true,
  "tags": [{ "name": "development" }]
}
```

##### Start a time entry (timer)

```
POST /team/{team_id}/time_entries/start
```

**Request Body:**

```json
{
  "tid": "abc123",
  "description": "Working on task",
  "billable": true
}
```

##### Stop the running timer

```
POST /team/{team_id}/time_entries/stop
```

##### Delete a time entry

```
DELETE /team/{team_id}/time_entries/{timer_id}
```

##### Update a time entry

```
PUT /team/{team_id}/time_entries/{timer_id}
```

### 6.6 Webhooks / Real-time

ClickUp supports webhooks at the workspace level.

**Create a webhook:**

```
POST /team/{team_id}/webhook
```

**Request Body:**

```json
{
  "endpoint": "https://esmer.example.com/webhooks/clickup",
  "events": [
    "taskCreated",
    "taskUpdated",
    "taskDeleted",
    "taskStatusUpdated",
    "taskAssigneeUpdated",
    "taskDueDateUpdated",
    "taskCommentPosted",
    "taskTimeTrackedUpdated"
  ]
}
```

**Available Events:**

| Event | Description |
|---|---|
| `taskCreated` | Task created |
| `taskUpdated` | Task updated |
| `taskDeleted` | Task deleted |
| `taskStatusUpdated` | Task status changed |
| `taskAssigneeUpdated` | Task assignee changed |
| `taskDueDateUpdated` | Task due date changed |
| `taskTagUpdated` | Task tag changed |
| `taskMoved` | Task moved to different list |
| `taskCommentPosted` | Comment added to task |
| `taskCommentUpdated` | Comment updated |
| `taskTimeEstimateUpdated` | Time estimate changed |
| `taskTimeTrackedUpdated` | Time tracking changed |
| `listCreated` | List created |
| `listUpdated` | List updated |
| `listDeleted` | List deleted |
| `folderCreated` | Folder created |
| `folderUpdated` | Folder updated |
| `folderDeleted` | Folder deleted |
| `spaceCreated` | Space created |
| `spaceUpdated` | Space updated |
| `spaceDeleted` | Space deleted |
| `goalCreated` | Goal created |
| `goalUpdated` | Goal updated |
| `goalDeleted` | Goal deleted |
| `keyResultCreated` | Key result created |
| `keyResultUpdated` | Key result updated |
| `keyResultDeleted` | Key result deleted |

**Webhook Payload:**

```json
{
  "event": "taskCreated",
  "history_items": [
    {
      "id": "hist_123",
      "type": 1,
      "date": "1707465600000",
      "field": "status",
      "parent_id": "abc123",
      "data": {},
      "user": { "id": 123, "username": "janedoe" }
    }
  ],
  "task_id": "abc123",
  "webhook_id": "wh_123"
}
```

### 6.7 Error Handling

| HTTP Code | Description | Esmer Action |
|---|---|---|
| 400 | Bad request | Validate request body |
| 401 | Unauthorized | Re-authenticate |
| 403 | Forbidden | Check workspace/list permissions |
| 404 | Not found | Verify resource IDs |
| 429 | Rate limited | Wait 60 seconds minimum; exponential backoff |
| 500 | Server error | Retry with exponential backoff |
| 503 | Service unavailable | Retry with exponential backoff |

### 6.8 Mobile-Specific Notes

- **Deep Hierarchy:** ClickUp has a 5-level hierarchy (Workspace > Space > Folder > List > Task). Esmer should cache this hierarchy and let users navigate it simply (e.g., "add task to [list name]").
- **Unix Timestamps:** All dates in ClickUp are Unix timestamps in milliseconds. Convert appropriately when displaying on mobile.
- **Custom Fields:** Fetch list custom fields before task creation to know available fields and their types.
- **Rate Limits:** At 100 req/min, ClickUp has the strictest rate limit of all services in this category. Implement aggressive request batching and caching.
- **Large Responses:** Task objects can be very large (custom fields, checklists, assignees). Use query parameters to limit fields when possible.
- **Deep Linking:** `https://app.clickup.com/t/{task_id}` opens a task in the ClickUp app (universal link support).
- **Timer Feature:** The start/stop timer endpoints are useful for Esmer's time tracking delegation feature.

---

## 7. Linear (Linear GraphQL API)

### 7.1 Service Overview

Linear is a modern issue tracking tool designed for software development teams, emphasizing speed and keyboard-driven workflows. It supports issues, projects, cycles (sprints), labels, comments, and integrations. Esmer will use Linear for engineering-focused users who need to create issues, update statuses, add comments, and manage their issue backlog from mobile.

### 7.2 Authentication

**Methods:** OAuth 2.0, Personal API Key

| Parameter | Value |
|---|---|
| Authorization URL | `https://linear.app/oauth/authorize` |
| Token URL | `https://api.linear.app/oauth/token` |
| Grant Type | `authorization_code` |
| Refresh Token Support | Yes |

**OAuth 2.0 Scopes:**

| Scope | Description | Required |
|---|---|---|
| `read` | Read access to all user data | Yes |
| `write` | Write access to create/update resources | Yes |
| `issues:create` | Create issues only | Optional (more granular) |
| `comments:create` | Create comments only | Optional |
| `admin` | Admin access (manage webhooks, integrations) | Optional (needed for webhook management) |

**OAuth Actor Setting:**
- **User** (default): Actions are performed as the authorizing user.
- **Application**: Actions are performed as the application itself.
- For Esmer, use **User** actor so tasks appear as created by the user.

**Personal API Key:**
- Generated at Linear Settings > Security & access > Personal API keys.
- Passed via `Authorization: {api_key}` header.

**Recommendations for Esmer:**
- Use OAuth 2.0 with `read` and `write` scopes for full issue management.
- Include `admin` scope if Esmer needs to manage webhooks.
- Store refresh tokens securely; use them to renew access tokens.
- Set the actor to **User** so that issues and comments appear under the user's name.

### 7.3 Base URL

```
https://api.linear.app/graphql
```

All requests are `POST` to this single endpoint with a JSON body containing the GraphQL `query` and optional `variables`.

### 7.4 Rate Limits

| Limit | Value |
|---|---|
| Complexity budget | 1,500 points per minute per authenticated user/application |
| Request budget | 250 requests per minute (separate from complexity) |
| Nested pagination limit | Max 50 items per connection in a single query |

Each query has a computed complexity cost based on the fields and connections requested. The `X-Complexity` response header reports the cost. Rate limit status is returned in `X-RateLimit-*` headers.

| Header | Description |
|---|---|
| `X-RateLimit-Requests-Remaining` | Requests remaining |
| `X-RateLimit-Requests-Reset` | Time until request limit resets (ISO 8601) |
| `X-RateLimit-Complexity-Remaining` | Complexity points remaining |
| `X-RateLimit-Complexity-Reset` | Time until complexity limit resets |

### 7.5 API Endpoints (GraphQL Queries and Mutations)

All operations use `POST https://api.linear.app/graphql` with:

```
Headers:
  Content-Type: application/json
  Authorization: Bearer {token}
```

#### 7.5.1 Issues

##### Query: Get issues

```graphql
query GetIssues($first: Int, $after: String, $filter: IssueFilter) {
  issues(first: $first, after: $after, filter: $filter) {
    pageInfo {
      hasNextPage
      endCursor
    }
    nodes {
      id
      identifier
      title
      description
      descriptionData
      priority
      priorityLabel
      estimate
      state {
        id
        name
        color
        type
      }
      assignee {
        id
        name
        email
        avatarUrl
      }
      creator {
        id
        name
      }
      team {
        id
        name
        key
      }
      project {
        id
        name
      }
      cycle {
        id
        name
        number
      }
      labels {
        nodes {
          id
          name
          color
        }
      }
      parent {
        id
        identifier
        title
      }
      children {
        nodes {
          id
          identifier
          title
        }
      }
      comments {
        nodes {
          id
          body
          createdAt
          user {
            id
            name
          }
        }
      }
      dueDate
      createdAt
      updatedAt
      completedAt
      canceledAt
      url
      branchName
    }
  }
}
```

**Variables:**

```json
{
  "first": 50,
  "after": null,
  "filter": {
    "assignee": { "id": { "eq": "user-uuid" } },
    "state": { "type": { "nin": ["completed", "canceled"] } },
    "team": { "key": { "eq": "ENG" } }
  }
}
```

**Filter Operators:** `eq`, `neq`, `in`, `nin`, `lt`, `lte`, `gt`, `gte`, `contains`, `notContains`, `startsWith`, `endsWith`, `and`, `or`.

##### Query: Get a single issue

```graphql
query GetIssue($id: String!) {
  issue(id: $id) {
    id
    identifier
    title
    description
    priority
    priorityLabel
    estimate
    state {
      id
      name
      color
      type
    }
    assignee {
      id
      name
      email
    }
    team {
      id
      name
      key
    }
    project {
      id
      name
    }
    labels {
      nodes {
        id
        name
        color
      }
    }
    dueDate
    createdAt
    updatedAt
    url
  }
}
```

**Variables:**

```json
{
  "id": "issue-uuid"
}
```

##### Mutation: Create an issue

```graphql
mutation CreateIssue($input: IssueCreateInput!) {
  issueCreate(input: $input) {
    success
    issue {
      id
      identifier
      title
      url
      state {
        id
        name
      }
    }
  }
}
```

**Variables:**

```json
{
  "input": {
    "title": "Fix login bug",
    "description": "Users report being logged out after 5 minutes",
    "teamId": "team-uuid",
    "assigneeId": "user-uuid",
    "stateId": "state-uuid",
    "priority": 2,
    "estimate": 3,
    "dueDate": "2026-02-15",
    "labelIds": ["label-uuid-1", "label-uuid-2"],
    "projectId": "project-uuid",
    "cycleId": "cycle-uuid",
    "parentId": null
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `title` | string | Yes | Issue title |
| `description` | string | No | Markdown description |
| `teamId` | string | Yes | Team ID |
| `assigneeId` | string | No | Assignee user ID |
| `stateId` | string | No | Workflow state ID |
| `priority` | integer | No | 0=none, 1=urgent, 2=high, 3=medium, 4=low |
| `estimate` | integer | No | Story point estimate |
| `dueDate` | string | No | Due date (YYYY-MM-DD) |
| `labelIds` | array[string] | No | Label IDs to apply |
| `projectId` | string | No | Project ID |
| `cycleId` | string | No | Cycle (sprint) ID |
| `parentId` | string | No | Parent issue ID (creates sub-issue) |

##### Mutation: Update an issue

```graphql
mutation UpdateIssue($id: String!, $input: IssueUpdateInput!) {
  issueUpdate(id: $id, input: $input) {
    success
    issue {
      id
      identifier
      title
      state {
        id
        name
      }
    }
  }
}
```

**Variables:**

```json
{
  "id": "issue-uuid",
  "input": {
    "stateId": "done-state-uuid",
    "priority": 1,
    "assigneeId": "new-user-uuid"
  }
}
```

##### Mutation: Delete an issue

```graphql
mutation DeleteIssue($id: String!) {
  issueDelete(id: $id) {
    success
  }
}
```

##### Mutation: Add a link to an issue

```graphql
mutation AttachmentLinkURL($issueId: String!, $url: String!, $title: String) {
  attachmentLinkURL(issueId: $issueId, url: $url, title: $title) {
    success
    attachment {
      id
      url
      title
    }
  }
}
```

#### 7.5.2 Comments

##### Query: Get comments on an issue

```graphql
query GetIssueComments($issueId: String!, $first: Int) {
  issue(id: $issueId) {
    comments(first: $first) {
      nodes {
        id
        body
        createdAt
        updatedAt
        user {
          id
          name
          avatarUrl
        }
      }
    }
  }
}
```

##### Mutation: Create a comment

```graphql
mutation CreateComment($input: CommentCreateInput!) {
  commentCreate(input: $input) {
    success
    comment {
      id
      body
      createdAt
    }
  }
}
```

**Variables:**

```json
{
  "input": {
    "issueId": "issue-uuid",
    "body": "Investigating this issue now. Delegated via Esmer."
  }
}
```

#### 7.5.3 Teams

##### Query: Get teams

```graphql
query GetTeams {
  teams {
    nodes {
      id
      name
      key
      description
      states {
        nodes {
          id
          name
          color
          type
          position
        }
      }
      labels {
        nodes {
          id
          name
          color
        }
      }
      members {
        nodes {
          id
          name
          email
          avatarUrl
        }
      }
    }
  }
}
```

#### 7.5.4 Projects

##### Query: Get projects

```graphql
query GetProjects($first: Int, $filter: ProjectFilter) {
  projects(first: $first, filter: $filter) {
    nodes {
      id
      name
      description
      state
      progress
      startDate
      targetDate
      teams {
        nodes {
          id
          name
        }
      }
      lead {
        id
        name
      }
      members {
        nodes {
          id
          name
        }
      }
      url
    }
  }
}
```

#### 7.5.5 Cycles (Sprints)

##### Query: Get active cycle

```graphql
query GetActiveCycle($teamId: String!) {
  team(id: $teamId) {
    activeCycle {
      id
      name
      number
      startsAt
      endsAt
      progress
      issues {
        nodes {
          id
          identifier
          title
          state {
            name
          }
        }
      }
    }
  }
}
```

#### 7.5.6 Users

##### Query: Get current user (viewer)

```graphql
query GetViewer {
  viewer {
    id
    name
    email
    avatarUrl
    organization {
      id
      name
    }
    teamMemberships {
      nodes {
        team {
          id
          name
          key
        }
      }
    }
  }
}
```

#### 7.5.7 Workflow States

##### Query: Get workflow states for a team

```graphql
query GetWorkflowStates($teamId: String!) {
  team(id: $teamId) {
    states {
      nodes {
        id
        name
        color
        type
        position
      }
    }
  }
}
```

State types: `triage`, `backlog`, `unstarted`, `started`, `completed`, `canceled`.

#### 7.5.8 Labels

##### Query: Get labels

```graphql
query GetLabels($first: Int) {
  issueLabels(first: $first) {
    nodes {
      id
      name
      color
      parent {
        id
        name
      }
    }
  }
}
```

##### Mutation: Create a label

```graphql
mutation CreateLabel($input: IssueLabelCreateInput!) {
  issueLabelCreate(input: $input) {
    success
    issueLabel {
      id
      name
      color
    }
  }
}
```

**Variables:**

```json
{
  "input": {
    "name": "mobile",
    "color": "#4194f6",
    "teamId": "team-uuid"
  }
}
```

### 7.6 Webhooks / Real-time

Linear supports webhooks for real-time notifications.

**Create a webhook (via API):**

```graphql
mutation CreateWebhook($input: WebhookCreateInput!) {
  webhookCreate(input: $input) {
    success
    webhook {
      id
      enabled
    }
  }
}
```

**Variables:**

```json
{
  "input": {
    "url": "https://esmer.example.com/webhooks/linear",
    "teamId": "team-uuid",
    "resourceTypes": ["Issue", "Comment", "Project", "Cycle"],
    "enabled": true,
    "secret": "webhook-signing-secret"
  }
}
```

**Supported Resource Types:**

| Resource | Actions |
|---|---|
| `Issue` | create, update, remove |
| `Comment` | create, update, remove |
| `Project` | create, update, remove |
| `Cycle` | create, update, remove |
| `IssueLabel` | create, update, remove |
| `Reaction` | create, remove |

**Webhook Payload:**

```json
{
  "action": "create",
  "type": "Issue",
  "createdAt": "2026-02-09T12:00:00.000Z",
  "data": {
    "id": "issue-uuid",
    "identifier": "ENG-123",
    "title": "Fix login bug",
    "priority": 2,
    "teamId": "team-uuid",
    "assigneeId": "user-uuid",
    "stateId": "state-uuid"
  },
  "url": "https://linear.app/workspace/issue/ENG-123",
  "organizationId": "org-uuid"
}
```

**Verification:** Sign the webhook with the `secret` provided during creation. Verify using HMAC-SHA256 of the raw body.

### 7.7 Error Handling

GraphQL errors are returned in the `errors` array of the response (HTTP 200):

```json
{
  "errors": [
    {
      "message": "Entity not found",
      "extensions": {
        "type": "NotFoundError",
        "userPresentableMessage": "The issue you're looking for doesn't exist"
      }
    }
  ],
  "data": null
}
```

| Error Type | Description | Esmer Action |
|---|---|---|
| `AuthenticationError` | Invalid or expired token | Refresh token; re-authenticate |
| `ForbiddenError` | Insufficient permissions | Check scopes and team membership |
| `NotFoundError` | Resource not found | Verify IDs |
| `ValidationError` | Invalid input | Validate input fields |
| `RateLimitError` | Rate limit exceeded | Wait per `X-RateLimit-*` headers |
| `InternalError` | Linear server error | Retry with exponential backoff |

HTTP-level errors:

| HTTP Code | Description | Esmer Action |
|---|---|---|
| 400 | Malformed GraphQL query | Fix query syntax |
| 401 | Authentication failed | Refresh/re-authenticate |
| 429 | Rate limited | Respect `Retry-After` header |
| 500 | Server error | Retry |

### 7.8 Mobile-Specific Notes

- **GraphQL Efficiency:** Request only the fields Esmer needs. Avoid deeply nested queries to stay within complexity limits.
- **Identifier Format:** Linear issues use human-readable identifiers like `ENG-123`. Always display these to users rather than UUIDs.
- **Pagination:** Use cursor-based pagination (`first`/`after`) with a page size of 25-50 for mobile.
- **Caching:** Cache team workflow states, labels, and members locally. These change infrequently.
- **Deep Linking:** `https://linear.app/{workspace}/issue/{identifier}` opens an issue in the Linear app (universal link support).
- **Priority Mapping:** Priority 0=none, 1=urgent, 2=high, 3=medium, 4=low. Display human-readable labels from `priorityLabel`.
- **Offline Strategy:** Queue issue creation and comment mutations. Linear's idempotent mutations make retry logic straightforward.
- **Markdown Support:** Issue descriptions and comments use Markdown. Esmer can pass Markdown directly from user input.

---

## 8. Trello (Trello REST API)

### 8.1 Service Overview

Trello is a visual project management tool using a board-list-card metaphor. Boards contain lists (columns), and lists contain cards (tasks). Cards support descriptions, checklists, labels, attachments, comments, due dates, and members. Esmer will use Trello for users who organize work visually on boards, enabling card creation, movement between lists, checklist management, and commenting from mobile.

### 8.2 Authentication

**Method:** API Key + Token

| Parameter | Value |
|---|---|
| Authentication URL | `https://trello.com/1/authorize?expiration=never&name=Esmer&scope=read,write&response_type=token&key={apiKey}` |
| Grant Type | User authorizes via browser, receives token |
| Token Expiration | Configurable (`1hour`, `1day`, `30days`, `never`) |

**Required Credentials:**

| Credential | Description | How to Obtain |
|---|---|---|
| API Key | Application identifier | Trello Power-Up Admin Portal > API Key |
| API Token | User authorization token | Generated via authorization URL above |

**Authentication Method:** All requests include `key` and `token` as query parameters:

```
GET https://api.trello.com/1/boards/{id}?key={apiKey}&token={apiToken}
```

**Scopes (set during authorization):**

| Scope | Description |
|---|---|
| `read` | Read access to boards, lists, cards |
| `write` | Write access (create, update, delete) |
| `account` | Read access to member account info |

**Recommendations for Esmer:**
- Request `read,write` scopes for full card management.
- Set token expiration to `never` for persistent access, or `30days` for more security.
- Trello does not use OAuth 2.0; it uses its own token-based system.
- Store both the API Key (app-level) and user Token (per-user) securely.
- The API Key is the same for all users of Esmer; the Token is per-user.

### 8.3 Base URL

```
https://api.trello.com/1
```

### 8.4 Rate Limits

| Limit | Value |
|---|---|
| Per API Key | 300 requests per 10 seconds |
| Per API Token | 100 requests per 10 seconds |
| Effective per-user limit | 100 requests per 10 seconds (token is the bottleneck) |

Rate limit responses return HTTP `429`. Trello includes `Retry-After` header. The per-token limit is the practical constraint for Esmer.

### 8.5 API Endpoints

#### 8.5.1 Boards

##### Get a board

```
GET /boards/{boardId}
```

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `boardId` | string | Yes | Board ID |

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `fields` | string | No | Comma-separated fields (e.g., `name,desc,url`) |
| `lists` | string | No | `all`, `open`, `closed`, `none` |
| `cards` | string | No | `all`, `visible`, `open`, `closed`, `none` |
| `members` | string | No | `all`, `normal`, `admins`, `owners`, `none` |
| `labels` | string | No | `all` or `none` |

**Response:**

```json
{
  "id": "5abbe4b7ddc1b351ef961414",
  "name": "Product Roadmap",
  "desc": "Our product roadmap board",
  "descData": null,
  "closed": false,
  "url": "https://trello.com/b/abc123/product-roadmap",
  "shortUrl": "https://trello.com/b/abc123",
  "prefs": {
    "permissionLevel": "org",
    "hideVotes": false,
    "voting": "disabled",
    "comments": "members",
    "selfJoin": true,
    "cardCovers": true,
    "background": "blue"
  },
  "labelNames": {
    "green": "Done",
    "yellow": "In Progress",
    "orange": "Blocked",
    "red": "Urgent",
    "purple": "Design",
    "blue": "Backend",
    "sky": "",
    "lime": "",
    "pink": "",
    "black": ""
  }
}
```

##### Create a board

```
POST /boards
```

**Query Parameters (used as body):**

| Name | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Board name |
| `desc` | string | No | Board description |
| `defaultLists` | boolean | No | Create default lists (To Do, Doing, Done) |
| `defaultLabels` | boolean | No | Create default labels |
| `idOrganization` | string | No | Organization/workspace ID |
| `prefs_permissionLevel` | string | No | `private`, `org`, `public` |
| `prefs_background` | string | No | Background color or image ID |

##### Update a board

```
PUT /boards/{boardId}
```

##### Delete a board

```
DELETE /boards/{boardId}
```

#### 8.5.2 Board Members

##### Get board members

```
GET /boards/{boardId}/members
```

##### Add member to board

```
PUT /boards/{boardId}/members/{memberId}
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | `admin`, `normal`, `observer` |

##### Invite member to board via email

```
PUT /boards/{boardId}/members
```

**Request Body:**

```json
{
  "email": "jane@example.com",
  "type": "normal"
}
```

##### Remove member from board

```
DELETE /boards/{boardId}/members/{memberId}
```

#### 8.5.3 Lists

##### Get lists on a board

```
GET /boards/{boardId}/lists
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `filter` | string | No | `all`, `open`, `closed` |
| `fields` | string | No | Comma-separated fields |

**Response:**

```json
[
  {
    "id": "5abbe4b7ddc1b351ef961415",
    "name": "To Do",
    "closed": false,
    "pos": 16384,
    "idBoard": "5abbe4b7ddc1b351ef961414"
  },
  {
    "id": "5abbe4b7ddc1b351ef961416",
    "name": "In Progress",
    "closed": false,
    "pos": 32768,
    "idBoard": "5abbe4b7ddc1b351ef961414"
  },
  {
    "id": "5abbe4b7ddc1b351ef961417",
    "name": "Done",
    "closed": false,
    "pos": 49152,
    "idBoard": "5abbe4b7ddc1b351ef961414"
  }
]
```

##### Create a list

```
POST /lists
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | List name |
| `idBoard` | string | Yes | Board ID |
| `pos` | string | No | `top`, `bottom`, or a positive float |

##### Update a list

```
PUT /lists/{listId}
```

##### Archive/unarchive a list

```
PUT /lists/{listId}/closed
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `value` | boolean | Yes | `true` to archive, `false` to unarchive |

##### Get cards in a list

```
GET /lists/{listId}/cards
```

#### 8.5.4 Cards

##### Get a card

```
GET /cards/{cardId}
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `fields` | string | No | Comma-separated fields |
| `attachments` | boolean | No | Include attachments |
| `members` | boolean | No | Include member objects |
| `checklists` | string | No | `all` or `none` |
| `checkItemStates` | boolean | No | Include check item states |

**Response:**

```json
{
  "id": "5abbe4b7ddc1b351ef961418",
  "name": "Implement login feature",
  "desc": "Build OAuth login flow for mobile app",
  "descData": null,
  "closed": false,
  "pos": 16384,
  "due": "2026-02-15T00:00:00.000Z",
  "dueComplete": false,
  "dueReminder": -1440,
  "start": "2026-02-09T00:00:00.000Z",
  "idBoard": "5abbe4b7ddc1b351ef961414",
  "idList": "5abbe4b7ddc1b351ef961416",
  "idMembers": ["member1"],
  "idLabels": ["label1", "label2"],
  "labels": [
    { "id": "label1", "idBoard": "...", "name": "Backend", "color": "blue" },
    { "id": "label2", "idBoard": "...", "name": "Urgent", "color": "red" }
  ],
  "url": "https://trello.com/c/abc123/1-implement-login-feature",
  "shortUrl": "https://trello.com/c/abc123",
  "idChecklists": ["cl1"],
  "checklists": [
    {
      "id": "cl1",
      "name": "Steps",
      "checkItems": [
        { "id": "ci1", "name": "Design OAuth flow", "state": "complete", "pos": 16384 },
        { "id": "ci2", "name": "Implement backend", "state": "incomplete", "pos": 32768 }
      ]
    }
  ],
  "badges": {
    "comments": 3,
    "attachments": 1,
    "checkItems": 2,
    "checkItemsChecked": 1
  },
  "cover": {
    "idAttachment": null,
    "color": null,
    "idUploadedBackground": null
  },
  "dateLastActivity": "2026-02-09T12:00:00.000Z"
}
```

##### Create a card

```
POST /cards
```

**Query Parameters (used as request params):**

| Name | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Card title |
| `idList` | string | Yes | List ID to add the card to |
| `desc` | string | No | Markdown description |
| `pos` | string | No | `top`, `bottom`, or positive float |
| `due` | string | No | Due date (ISO 8601) |
| `start` | string | No | Start date |
| `dueComplete` | boolean | No | Whether due date is complete |
| `dueReminder` | integer | No | Reminder minutes before due (-1440 = 1 day) |
| `idMembers` | string | No | Comma-separated member IDs |
| `idLabels` | string | No | Comma-separated label IDs |
| `urlSource` | string | No | URL to attach to card |
| `idCardSource` | string | No | Card ID to copy from |
| `keepFromSource` | string | No | Fields to keep from source card |

##### Update a card

```
PUT /cards/{cardId}
```

**Common update operations:**

Move card to a different list:
```
PUT /cards/{cardId}?idList={newListId}
```

Mark as complete:
```
PUT /cards/{cardId}?dueComplete=true
```

Change position:
```
PUT /cards/{cardId}?pos=top
```

##### Delete a card

```
DELETE /cards/{cardId}
```

#### 8.5.5 Card Comments

##### Get comments on a card

```
GET /cards/{cardId}/actions
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `filter` | string | No | `commentCard` to get only comments |

**Response:**

```json
[
  {
    "id": "action1",
    "type": "commentCard",
    "date": "2026-02-09T12:00:00.000Z",
    "data": {
      "text": "Looking into this issue",
      "card": { "id": "card1", "name": "Fix bug" }
    },
    "memberCreator": {
      "id": "member1",
      "username": "janedoe",
      "fullName": "Jane Doe"
    }
  }
]
```

##### Create a comment on a card

```
POST /cards/{cardId}/actions/comments
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `text` | string | Yes | Comment text (Markdown supported) |

##### Update a comment

```
PUT /cards/{cardId}/actions/{actionId}/comments
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `text` | string | Yes | Updated comment text |

##### Delete a comment

```
DELETE /cards/{cardId}/actions/{actionId}/comments
```

#### 8.5.6 Checklists

##### Get checklists on a card

```
GET /cards/{cardId}/checklists
```

##### Create a checklist

```
POST /cards/{cardId}/checklists
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `name` | string | No | Checklist name |
| `idChecklistSource` | string | No | Checklist to copy from |
| `pos` | string | No | Position |

##### Delete a checklist

```
DELETE /checklists/{checklistId}
```

##### Create a checklist item

```
POST /checklists/{checklistId}/checkItems
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Item name |
| `pos` | string | No | Position |
| `checked` | boolean | No | Initial checked state |
| `due` | string | No | Due date |
| `idMember` | string | No | Assigned member |

##### Update a checklist item

```
PUT /cards/{cardId}/checkItem/{checkItemId}
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `name` | string | No | Updated name |
| `state` | string | No | `complete` or `incomplete` |
| `pos` | string | No | Position |
| `due` | string | No | Due date |
| `idMember` | string | No | Member ID |

##### Delete a checklist item

```
DELETE /checklists/{checklistId}/checkItems/{checkItemId}
```

#### 8.5.7 Labels

##### Get labels on a board

```
GET /boards/{boardId}/labels
```

**Response:**

```json
[
  {
    "id": "label1",
    "idBoard": "board1",
    "name": "Urgent",
    "color": "red"
  }
]
```

##### Create a label

```
POST /boards/{boardId}/labels
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Label name |
| `color` | string | Yes | `green`, `yellow`, `orange`, `red`, `purple`, `blue`, `sky`, `lime`, `pink`, `black`, or `null` |

##### Update a label

```
PUT /labels/{labelId}
```

##### Delete a label

```
DELETE /labels/{labelId}
```

##### Add a label to a card

```
POST /cards/{cardId}/idLabels
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `value` | string | Yes | Label ID |

##### Remove a label from a card

```
DELETE /cards/{cardId}/idLabels/{labelId}
```

#### 8.5.8 Attachments

##### Get attachments on a card

```
GET /cards/{cardId}/attachments
```

**Response:**

```json
[
  {
    "id": "att1",
    "name": "screenshot.png",
    "url": "https://trello.com/1/cards/card1/attachments/att1/download/screenshot.png",
    "bytes": 102400,
    "date": "2026-02-09T12:00:00.000Z",
    "mimeType": "image/png",
    "isUpload": true
  }
]
```

##### Create an attachment

```
POST /cards/{cardId}/attachments
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `name` | string | No | Attachment name |
| `url` | string | Conditional | URL to attach (if not uploading file) |
| `file` | file | Conditional | File to upload (multipart) |
| `mimeType` | string | No | MIME type |

##### Delete an attachment

```
DELETE /cards/{cardId}/attachments/{attachmentId}
```

### 8.6 Webhooks / Real-time

Trello supports webhooks on models (boards, lists, cards, members).

**Create a webhook:**

```
POST /webhooks
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `callbackURL` | string | Yes | Your webhook endpoint |
| `idModel` | string | Yes | ID of the model to watch (board, list, card, or member) |
| `description` | string | No | Description |
| `active` | boolean | No | Whether the webhook is active (default true) |

**Verification:** Trello sends a `HEAD` request to your `callbackURL` on creation. You must respond with `200 OK`.

**Subsequent Notifications:** `POST` to your callback URL.

**Webhook Payload:**

```json
{
  "action": {
    "id": "action1",
    "type": "updateCard",
    "date": "2026-02-09T12:00:00.000Z",
    "data": {
      "card": { "id": "card1", "name": "Fix bug", "idList": "list2" },
      "listBefore": { "id": "list1", "name": "To Do" },
      "listAfter": { "id": "list2", "name": "In Progress" },
      "board": { "id": "board1", "name": "Product Roadmap" },
      "old": { "idList": "list1" }
    },
    "memberCreator": {
      "id": "member1",
      "username": "janedoe",
      "fullName": "Jane Doe"
    }
  },
  "model": {
    "id": "board1",
    "name": "Product Roadmap"
  }
}
```

**Action Types:**

| Action Type | Description |
|---|---|
| `createCard` | Card created |
| `updateCard` | Card updated (moved, renamed, etc.) |
| `deleteCard` | Card deleted |
| `addMemberToCard` | Member added to card |
| `removeMemberFromCard` | Member removed from card |
| `commentCard` | Comment added to card |
| `addAttachmentToCard` | Attachment added |
| `addChecklistToCard` | Checklist added |
| `updateCheckItemStateOnCard` | Checklist item checked/unchecked |
| `createList` | List created |
| `updateList` | List updated |
| `moveCardToBoard` | Card moved to different board |
| `addLabelToCard` | Label added to card |
| `removeLabelFromCard` | Label removed from card |

**Verification:** Validate webhooks using HMAC-SHA1 of the request body + callbackURL, using your API secret. Check the `x-trello-webhook` header.

### 8.7 Error Handling

| HTTP Code | Description | Esmer Action |
|---|---|---|
| 400 | Bad request (invalid parameters) | Validate input |
| 401 | Invalid key or token | Re-authenticate user |
| 403 | Insufficient permissions | Verify board/card access |
| 404 | Resource not found | Verify IDs |
| 429 | Rate limited | Wait per `Retry-After` header |
| 449 | Retry | Trello-specific: retry the request |
| 500 | Server error | Retry with exponential backoff |

### 8.8 Mobile-Specific Notes

- **Visual Model:** Trello's board-list-card model maps well to mobile UI. Esmer can show boards as a list, lists as columns, and cards as items.
- **Query Parameter Auth:** Unlike other services, auth credentials are passed as query parameters, not headers. Ensure tokens do not appear in logs.
- **Card Movement:** The most common Esmer action will be moving cards between lists (e.g., "move 'Fix bug' to In Progress"). Use `PUT /cards/{id}?idList={newListId}`.
- **Compact Responses:** Use `fields` parameter to limit returned fields and reduce payload size.
- **Batch API:** Trello supports batch requests: `GET /batch?urls=/cards/id1,/cards/id2`. Use this to reduce request count when loading multiple cards.
- **Deep Linking:** `trello://trello.com/c/{shortLink}` opens a card in the Trello mobile app. Also, `https://trello.com/c/{shortLink}` works as a universal link.
- **No Expiring Tokens:** Tokens created with `expiration=never` persist indefinitely. Store securely in device keychain.
- **Webhook on Board:** Create webhooks on the board model to receive all card events. This is more efficient than per-card webhooks.

---

## 9. Jira Software (Atlassian REST API v3)

### 9.1 Service Overview

Jira Software is Atlassian's enterprise issue and project tracking platform used extensively in software development. It supports projects, issues (epics, stories, tasks, bugs, sub-tasks), sprints, boards, workflows, comments, attachments, labels, components, custom fields, and advanced querying via JQL (Jira Query Language). Esmer will use Jira for users in enterprise environments who need to create issues, update statuses, add comments, log work, and query their backlog from mobile.

### 9.2 Authentication

**Methods:** API Token (Cloud), Basic Auth (Server/Data Center), OAuth 2.0 (3LO for Cloud)

**Cloud - API Token (Recommended for Esmer):**

| Parameter | Value |
|---|---|
| Authentication | Basic Auth with email + API token |
| Header | `Authorization: Basic base64(email:api_token)` |
| Token Source | Atlassian account > Security > API Tokens |

**Cloud - OAuth 2.0 (3LO):**

| Parameter | Value |
|---|---|
| Authorization URL | `https://auth.atlassian.com/authorize` |
| Token URL | `https://auth.atlassian.com/oauth/token` |
| Grant Type | `authorization_code` |
| Refresh Token Support | Yes |
| Audience | `api.atlassian.com` |

**OAuth 2.0 Scopes (Jira Cloud):**

| Scope | Description | Required |
|---|---|---|
| `read:jira-work` | Read Jira project and issue data | Yes |
| `write:jira-work` | Create and edit issues, comments, worklogs | Yes |
| `read:jira-user` | Read user information | Recommended |
| `manage:jira-project` | Manage project settings | Optional |
| `manage:jira-configuration` | Manage Jira configuration | Optional |
| `offline_access` | Obtain refresh tokens | Yes |

**Server/Data Center - Basic Auth:**
- Uses email + password (not API token).
- `Authorization: Basic base64(email:password)`

**Recommendations for Esmer:**
- For Jira Cloud: Use OAuth 2.0 (3LO) for multi-user support, or API Token for simpler integration.
- Always include `offline_access` scope to get refresh tokens.
- After OAuth, call `GET https://api.atlassian.com/oauth/token/accessible-resources` to get the user's cloud instance IDs.
- API tokens do not expire; OAuth access tokens expire in 1 hour.
- For Jira Server/Data Center: Use Basic Auth with username and password.

### 9.3 Base URL

**Jira Cloud:**
```
https://{your-domain}.atlassian.net/rest/api/3
```

**Via OAuth 2.0 (3LO):**
```
https://api.atlassian.com/ex/jira/{cloudId}/rest/api/3
```

**Jira Server/Data Center:**
```
https://{your-server}/rest/api/2
```

### 9.4 Rate Limits

| Limit | Value |
|---|---|
| Jira Cloud (per user) | Varies; no fixed published limit |
| Concurrent requests | ~10 concurrent requests per user |
| Rate limit headers | `X-RateLimit-NearLimit` (warning), `Retry-After` (when throttled) |
| Throttle response | HTTP 429 |

Jira Cloud uses adaptive rate limiting. The `X-RateLimit-NearLimit` header appears as a warning before throttling occurs. Jira Server does not have explicit rate limiting.

### 9.5 API Endpoints

#### 9.5.1 Projects

##### Get all projects

```
GET /rest/api/3/project
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `startAt` | integer | No | Pagination offset (default 0) |
| `maxResults` | integer | No | Max results (default 50, max 50) |
| `orderBy` | string | No | `category`, `-category`, `key`, `-key`, `name`, `-name`, `owner`, `-owner` |
| `query` | string | No | Search by project name |
| `typeKey` | string | No | `software`, `service_desk`, `business` |
| `expand` | string | No | `description`, `lead`, `issueTypes`, `url`, `projectKeys`, `permissions`, `insight` |

**Response:**

```json
{
  "self": "https://your-domain.atlassian.net/rest/api/3/project",
  "maxResults": 50,
  "startAt": 0,
  "total": 2,
  "isLast": true,
  "values": [
    {
      "self": "https://your-domain.atlassian.net/rest/api/3/project/10001",
      "id": "10001",
      "key": "ENG",
      "name": "Engineering",
      "projectTypeKey": "software",
      "simplified": false,
      "avatarUrls": {},
      "lead": {
        "self": "...",
        "accountId": "5b10a2844c20165700ede21g",
        "displayName": "Jane Doe",
        "active": true
      },
      "issueTypes": [
        { "id": "10001", "name": "Story", "subtask": false },
        { "id": "10002", "name": "Bug", "subtask": false },
        { "id": "10003", "name": "Task", "subtask": false },
        { "id": "10004", "name": "Sub-task", "subtask": true },
        { "id": "10005", "name": "Epic", "subtask": false }
      ]
    }
  ]
}
```

##### Get a project

```
GET /rest/api/3/project/{projectIdOrKey}
```

#### 9.5.2 Issues

##### Search issues (JQL)

```
POST /rest/api/3/search
```

**Request Body:**

```json
{
  "jql": "project = ENG AND assignee = currentUser() AND status != Done ORDER BY priority DESC, created ASC",
  "startAt": 0,
  "maxResults": 50,
  "fields": [
    "summary",
    "status",
    "assignee",
    "reporter",
    "priority",
    "issuetype",
    "created",
    "updated",
    "duedate",
    "labels",
    "components",
    "description",
    "comment",
    "attachment",
    "timetracking",
    "customfield_10001"
  ],
  "expand": ["renderedFields", "names"]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `jql` | string | Yes | JQL query string |
| `startAt` | integer | No | Pagination offset |
| `maxResults` | integer | No | Max results (max 100 for Cloud, 1000 for Server) |
| `fields` | array[string] | No | Fields to return (use `["*all"]` for all, `["-description"]` to exclude) |
| `expand` | array[string] | No | Additional data to include |

**Common JQL queries for Esmer:**

| Use Case | JQL |
|---|---|
| My open tasks | `assignee = currentUser() AND status != Done` |
| Overdue issues | `assignee = currentUser() AND duedate < now() AND status != Done` |
| Today's issues | `assignee = currentUser() AND duedate = startOfDay()` |
| Sprint issues | `assignee = currentUser() AND sprint in openSprints()` |
| Recently updated | `project = ENG AND updated >= -1d ORDER BY updated DESC` |

**Response:**

```json
{
  "startAt": 0,
  "maxResults": 50,
  "total": 125,
  "issues": [
    {
      "id": "10002",
      "self": "https://your-domain.atlassian.net/rest/api/3/issue/10002",
      "key": "ENG-42",
      "fields": {
        "summary": "Fix login timeout bug",
        "status": {
          "self": "...",
          "id": "10001",
          "name": "In Progress",
          "statusCategory": {
            "id": 4,
            "key": "indeterminate",
            "name": "In Progress",
            "colorName": "blue"
          }
        },
        "assignee": {
          "accountId": "5b10a2844c20165700ede21g",
          "displayName": "Jane Doe",
          "emailAddress": "jane@example.com",
          "avatarUrls": {}
        },
        "reporter": {
          "accountId": "5b10a2844c20165700ede21h",
          "displayName": "John Smith"
        },
        "priority": {
          "id": "2",
          "name": "High",
          "iconUrl": "..."
        },
        "issuetype": {
          "id": "10002",
          "name": "Bug",
          "subtask": false
        },
        "created": "2026-02-01T10:00:00.000+0000",
        "updated": "2026-02-09T12:00:00.000+0000",
        "duedate": "2026-02-15",
        "labels": ["backend", "security"],
        "components": [
          { "id": "10000", "name": "Authentication" }
        ],
        "description": {
          "type": "doc",
          "version": 1,
          "content": [
            {
              "type": "paragraph",
              "content": [
                { "type": "text", "text": "Users are logged out after 5 minutes." }
              ]
            }
          ]
        },
        "comment": {
          "comments": [],
          "maxResults": 0,
          "total": 0,
          "startAt": 0
        },
        "timetracking": {
          "originalEstimate": "2h",
          "remainingEstimate": "1h",
          "timeSpent": "1h",
          "originalEstimateSeconds": 7200,
          "remainingEstimateSeconds": 3600,
          "timeSpentSeconds": 3600
        }
      }
    }
  ]
}
```

##### Get an issue

```
GET /rest/api/3/issue/{issueIdOrKey}
```

**Path Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `issueIdOrKey` | string | Yes | Issue ID (`10002`) or key (`ENG-42`) |

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `fields` | string | No | Comma-separated fields |
| `expand` | string | No | Additional data |

##### Create an issue

```
POST /rest/api/3/issue
```

**Request Body:**

```json
{
  "fields": {
    "project": {
      "key": "ENG"
    },
    "summary": "Fix login timeout bug",
    "description": {
      "type": "doc",
      "version": 1,
      "content": [
        {
          "type": "paragraph",
          "content": [
            { "type": "text", "text": "Users are being logged out after 5 minutes of inactivity." }
          ]
        }
      ]
    },
    "issuetype": {
      "name": "Bug"
    },
    "assignee": {
      "accountId": "5b10a2844c20165700ede21g"
    },
    "reporter": {
      "accountId": "5b10a2844c20165700ede21h"
    },
    "priority": {
      "name": "High"
    },
    "labels": ["backend", "security"],
    "components": [
      { "name": "Authentication" }
    ],
    "duedate": "2026-02-15",
    "timetracking": {
      "originalEstimate": "2h"
    },
    "parent": {
      "key": "ENG-40"
    },
    "customfield_10001": "custom value"
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `project` | object | Yes | Project key or ID |
| `summary` | string | Yes | Issue title |
| `issuetype` | object | Yes | Issue type name or ID |
| `description` | object | No | Atlassian Document Format (ADF) |
| `assignee` | object | No | Assignee account ID |
| `reporter` | object | No | Reporter account ID |
| `priority` | object | No | Priority name or ID |
| `labels` | array[string] | No | Label names |
| `components` | array[object] | No | Component names |
| `duedate` | string | No | Due date (YYYY-MM-DD) |
| `timetracking` | object | No | Time estimates |
| `parent` | object | No | Parent issue key (for sub-tasks) |

**Response:**

```json
{
  "id": "10002",
  "key": "ENG-42",
  "self": "https://your-domain.atlassian.net/rest/api/3/issue/10002"
}
```

##### Update an issue

```
PUT /rest/api/3/issue/{issueIdOrKey}
```

**Request Body:**

```json
{
  "fields": {
    "summary": "Updated summary",
    "priority": { "name": "Critical" },
    "labels": ["backend", "security", "urgent"]
  }
}
```

##### Delete an issue

```
DELETE /rest/api/3/issue/{issueIdOrKey}
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `deleteSubtasks` | boolean | No | Delete subtasks too (default false) |

##### Transition an issue (change status)

```
POST /rest/api/3/issue/{issueIdOrKey}/transitions
```

First, get available transitions:

```
GET /rest/api/3/issue/{issueIdOrKey}/transitions
```

**Response:**

```json
{
  "transitions": [
    {
      "id": "21",
      "name": "In Progress",
      "to": {
        "id": "10001",
        "name": "In Progress",
        "statusCategory": { "id": 4, "name": "In Progress" }
      }
    },
    {
      "id": "31",
      "name": "Done",
      "to": {
        "id": "10002",
        "name": "Done",
        "statusCategory": { "id": 3, "name": "Done" }
      }
    }
  ]
}
```

Then perform the transition:

```
POST /rest/api/3/issue/{issueIdOrKey}/transitions
```

**Request Body:**

```json
{
  "transition": {
    "id": "21"
  },
  "fields": {
    "resolution": {
      "name": "Done"
    }
  },
  "update": {
    "comment": [
      {
        "add": {
          "body": {
            "type": "doc",
            "version": 1,
            "content": [
              {
                "type": "paragraph",
                "content": [
                  { "type": "text", "text": "Moved to In Progress via Esmer" }
                ]
              }
            ]
          }
        }
      }
    ]
  }
}
```

##### Get issue changelog

```
GET /rest/api/3/issue/{issueIdOrKey}/changelog
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `startAt` | integer | No | Pagination offset |
| `maxResults` | integer | No | Max results |

##### Send notification for an issue

```
POST /rest/api/3/issue/{issueIdOrKey}/notify
```

**Request Body:**

```json
{
  "subject": "Issue update from Esmer",
  "textBody": "The issue has been updated",
  "to": {
    "reporter": true,
    "assignee": true,
    "watchers": true
  }
}
```

#### 9.5.3 Issue Comments

##### Get comments

```
GET /rest/api/3/issue/{issueIdOrKey}/comment
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `startAt` | integer | No | Offset |
| `maxResults` | integer | No | Max results |
| `orderBy` | string | No | `created`, `-created` |
| `expand` | string | No | `renderedBody` |

**Response:**

```json
{
  "startAt": 0,
  "maxResults": 50,
  "total": 3,
  "comments": [
    {
      "self": "...",
      "id": "10001",
      "author": {
        "accountId": "5b10a2844c20165700ede21g",
        "displayName": "Jane Doe"
      },
      "body": {
        "type": "doc",
        "version": 1,
        "content": [
          {
            "type": "paragraph",
            "content": [
              { "type": "text", "text": "Working on this now." }
            ]
          }
        ]
      },
      "created": "2026-02-09T12:00:00.000+0000",
      "updated": "2026-02-09T12:00:00.000+0000"
    }
  ]
}
```

##### Add a comment

```
POST /rest/api/3/issue/{issueIdOrKey}/comment
```

**Request Body:**

```json
{
  "body": {
    "type": "doc",
    "version": 1,
    "content": [
      {
        "type": "paragraph",
        "content": [
          { "type": "text", "text": "Comment added by Esmer on behalf of user." }
        ]
      }
    ]
  },
  "visibility": {
    "type": "role",
    "value": "Developers"
  }
}
```

##### Update a comment

```
PUT /rest/api/3/issue/{issueIdOrKey}/comment/{commentId}
```

##### Delete a comment

```
DELETE /rest/api/3/issue/{issueIdOrKey}/comment/{commentId}
```

#### 9.5.4 Issue Attachments

##### Get attachments

```
GET /rest/api/3/issue/{issueIdOrKey}?fields=attachment
```

##### Add an attachment

```
POST /rest/api/3/issue/{issueIdOrKey}/attachments
```

**Headers:**

| Name | Value |
|---|---|
| `X-Atlassian-Token` | `no-check` (required to bypass XSRF check) |
| `Content-Type` | `multipart/form-data` |

**Body:** Multipart form data with `file` field.

##### Get attachment metadata

```
GET /rest/api/3/attachment/{attachmentId}
```

##### Delete an attachment

```
DELETE /rest/api/3/attachment/{attachmentId}
```

#### 9.5.5 Users

##### Search users

```
GET /rest/api/3/user/search
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `query` | string | No | Search by display name or email |
| `accountId` | string | No | Specific account ID |
| `startAt` | integer | No | Offset |
| `maxResults` | integer | No | Max results (max 1000) |

##### Get current user

```
GET /rest/api/3/myself
```

##### Create a user

```
POST /rest/api/3/user
```

**Request Body:**

```json
{
  "emailAddress": "jane@example.com",
  "displayName": "Jane Doe",
  "products": ["jira-software"]
}
```

##### Delete a user

```
DELETE /rest/api/3/user
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `accountId` | string | Yes | Account ID to delete |

#### 9.5.6 Boards and Sprints (Agile API)

The Agile API uses a different base path:

```
https://{your-domain}.atlassian.net/rest/agile/1.0
```

##### Get all boards

```
GET /rest/agile/1.0/board
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `startAt` | integer | No | Offset |
| `maxResults` | integer | No | Max results (max 50) |
| `type` | string | No | `scrum`, `kanban`, `simple` |
| `name` | string | No | Filter by board name |
| `projectKeyOrId` | string | No | Filter by project |

##### Get sprints for a board

```
GET /rest/agile/1.0/board/{boardId}/sprint
```

**Query Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `state` | string | No | `future`, `active`, `closed` |
| `startAt` | integer | No | Offset |
| `maxResults` | integer | No | Max results |

**Response:**

```json
{
  "maxResults": 50,
  "startAt": 0,
  "values": [
    {
      "id": 1,
      "self": "...",
      "state": "active",
      "name": "Sprint 42",
      "startDate": "2026-02-03T00:00:00.000Z",
      "endDate": "2026-02-17T00:00:00.000Z",
      "originBoardId": 1,
      "goal": "Complete login feature"
    }
  ]
}
```

##### Get issues in a sprint

```
GET /rest/agile/1.0/sprint/{sprintId}/issue
```

##### Move issues to sprint

```
POST /rest/agile/1.0/sprint/{sprintId}/issue
```

**Request Body:**

```json
{
  "issues": ["ENG-42", "ENG-43"]
}
```

### 9.6 Webhooks / Real-time

Jira supports webhooks for real-time notifications.

**Register a webhook (Cloud via REST API):**

```
POST /rest/api/3/webhook
```

**Alternative: Register via Jira Admin UI** (System > WebHooks).

**Webhook Events:**

| Event | Description |
|---|---|
| `jira:issue_created` | Issue created |
| `jira:issue_updated` | Issue updated |
| `jira:issue_deleted` | Issue deleted |
| `comment_created` | Comment created |
| `comment_updated` | Comment updated |
| `comment_deleted` | Comment deleted |
| `attachment_created` | Attachment added |
| `attachment_deleted` | Attachment removed |
| `sprint_created` | Sprint created |
| `sprint_started` | Sprint started |
| `sprint_closed` | Sprint closed |
| `board_created` | Board created |
| `board_updated` | Board updated |
| `board_deleted` | Board deleted |
| `issuelink_created` | Issue link created |
| `issuelink_deleted` | Issue link deleted |
| `worklog_created` | Work log added |
| `worklog_updated` | Work log updated |
| `worklog_deleted` | Work log deleted |

**Webhook Payload (issue_updated example):**

```json
{
  "timestamp": 1707465600000,
  "webhookEvent": "jira:issue_updated",
  "issue_event_type_name": "issue_generic",
  "user": {
    "accountId": "5b10a2844c20165700ede21g",
    "displayName": "Jane Doe"
  },
  "issue": {
    "id": "10002",
    "key": "ENG-42",
    "fields": {
      "summary": "Fix login timeout bug",
      "status": { "name": "In Progress" },
      "assignee": { "displayName": "Jane Doe" },
      "priority": { "name": "High" }
    }
  },
  "changelog": {
    "items": [
      {
        "field": "status",
        "fieldtype": "jira",
        "from": "10000",
        "fromString": "To Do",
        "to": "10001",
        "toString": "In Progress"
      }
    ]
  }
}
```

### 9.7 Error Handling

| HTTP Code | Description | Esmer Action |
|---|---|---|
| 400 | Bad request (invalid JQL, missing fields) | Validate JQL syntax and required fields |
| 401 | Authentication failed | Refresh token or re-authenticate |
| 403 | Forbidden (insufficient permissions) | Check project/issue permissions |
| 404 | Issue/project not found | Verify issue key or project key |
| 405 | Method not allowed | Check endpoint and HTTP method |
| 409 | Conflict (concurrent edit) | Re-fetch and retry |
| 429 | Rate limited | Wait per `Retry-After` header |
| 500 | Server error | Retry with exponential backoff |

Jira error responses include detailed error messages:

```json
{
  "errorMessages": ["Issue does not exist or you do not have permission to see it."],
  "errors": {
    "summary": "You must specify a summary of the issue."
  }
}
```

### 9.8 Mobile-Specific Notes

- **Atlassian Document Format (ADF):** Jira v3 API uses ADF for descriptions and comments (structured JSON, not plain text or Markdown). Esmer must convert user text input to ADF format. For simple text, wrap in `{type: "doc", version: 1, content: [{type: "paragraph", content: [{type: "text", text: "..."}]}]}`.
- **Issue Keys:** Jira issues use project-prefix keys like `ENG-42`. Always display these to users.
- **JQL Power:** JQL is extremely powerful for filtering. Esmer should provide pre-built JQL templates for common queries (my tasks, overdue, sprint backlog).
- **Transitions:** To change an issue's status, you must first fetch available transitions, then execute the transition by ID. This is a two-step process.
- **Cloud ID:** When using OAuth 2.0, the cloud instance ID must be retrieved first via `GET https://api.atlassian.com/oauth/token/accessible-resources` before making API calls.
- **Large Payloads:** Jira responses can be very large. Always specify `fields` to limit returned data.
- **Deep Linking:** `https://{domain}.atlassian.net/browse/{issueKey}` opens an issue. The Jira mobile app handles universal links.
- **Offline Caveats:** Jira's workflow transitions and custom field requirements make offline issue creation complex. Consider queuing only comments and simple updates for offline, and requiring connectivity for issue creation.

---

## 10. monday.com (monday.com GraphQL API)

### 10.1 Service Overview

monday.com is a work management platform built around boards, groups, items, and columns. Boards represent projects, groups organize items within boards, items are individual work entries (tasks), and columns define the data schema (status, date, person, text, etc.). Esmer will use monday.com for teams that manage their workflows on monday.com, enabling item creation, status updates, column value changes, and update posting from mobile.

### 10.2 Authentication

**Methods:** OAuth 2.0, API Token (Personal/App)

| Parameter | Value |
|---|---|
| Authorization URL | `https://auth.monday.com/oauth2/authorize` |
| Token URL | `https://auth.monday.com/oauth2/token` |
| Grant Type | `authorization_code` |
| Refresh Token Support | No (monday.com tokens do not expire) |

**OAuth 2.0 Scopes:**

| Scope | Description | Required |
|---|---|---|
| `boards:read` | Read board data | Yes |
| `boards:write` | Write board data (create/update items) | Yes |
| `workspaces:read` | Read workspace data | Recommended |
| `workspaces:write` | Write workspace data | Optional |
| `users:read` | Read user data | Recommended |
| `account:read` | Read account info | Optional |
| `updates:read` | Read updates (comments) | Recommended |
| `updates:write` | Write updates | Recommended |
| `notifications:write` | Send notifications | Optional |

**API Token (Personal):**
- Generated from monday.com Developer Center > My Access Tokens.
- Passed via `Authorization: {token}` header.
- Full access to all resources the user can access.

**Recommendations for Esmer:**
- Use OAuth 2.0 for multi-user deployments with `boards:read`, `boards:write`, `users:read`, `updates:read`, and `updates:write` scopes.
- monday.com tokens do not expire, simplifying token management.
- API tokens require n8n version 1.22.6 or above (per n8n docs); Esmer does not have this constraint but should note the maturity of the API integration.

### 10.3 Base URL

```
https://api.monday.com/v2
```

All requests are `POST` to this single endpoint with a JSON body containing the GraphQL `query` and optional `variables`.

**Headers:**

| Header | Value |
|---|---|
| `Content-Type` | `application/json` |
| `Authorization` | `{token}` (no "Bearer" prefix) |
| `API-Version` | `2024-10` (recommended to pin API version) |

### 10.4 Rate Limits

| Limit | Value |
|---|---|
| Complexity budget | 5,000,000 points per minute per account (shared across all users) |
| Per-query complexity | Computed based on requested fields and pagination |
| Rate limit headers | `X-Complexity-Limit`, `X-Complexity-After`, `X-Complexity-Before` |
| Throttle response | Error in GraphQL response with `COMPLEXITY_BUDGET_EXHAUSTED` code |

Each query returns its complexity cost in the response:

```json
{
  "data": { ... },
  "account_id": 12345,
  "complexity": {
    "before": 4999000,
    "after": 4998500,
    "query": 500,
    "reset_in_x_seconds": 45
  }
}
```

### 10.5 API Endpoints (GraphQL Queries and Mutations)

All operations use `POST https://api.monday.com/v2` with a JSON body.

#### 10.5.1 Boards

##### Query: Get all boards

```graphql
query GetBoards($limit: Int, $page: Int) {
  boards(limit: $limit, page: $page) {
    id
    name
    description
    state
    board_kind
    owner {
      id
      name
      email
    }
    columns {
      id
      title
      type
      settings_str
    }
    groups {
      id
      title
      color
      position
    }
    workspace {
      id
      name
    }
    permissions
    updated_at
  }
}
```

**Variables:**

```json
{
  "limit": 25,
  "page": 1
}
```

**Response:**

```json
{
  "data": {
    "boards": [
      {
        "id": "1234567890",
        "name": "Product Roadmap",
        "description": "Q1 product roadmap",
        "state": "active",
        "board_kind": "public",
        "owner": {
          "id": "9876543",
          "name": "Jane Doe",
          "email": "jane@example.com"
        },
        "columns": [
          { "id": "name", "title": "Name", "type": "name", "settings_str": "{}" },
          { "id": "status", "title": "Status", "type": "status", "settings_str": "{\"labels\":{\"0\":\"Working on it\",\"1\":\"Done\",\"2\":\"Stuck\"}}" },
          { "id": "person", "title": "Owner", "type": "people", "settings_str": "{}" },
          { "id": "date4", "title": "Due Date", "type": "date", "settings_str": "{}" },
          { "id": "text0", "title": "Description", "type": "text", "settings_str": "{}" },
          { "id": "priority", "title": "Priority", "type": "status", "settings_str": "{\"labels\":{\"0\":\"Critical\",\"1\":\"High\",\"2\":\"Medium\",\"3\":\"Low\"}}" }
        ],
        "groups": [
          { "id": "new_group", "title": "To Do", "color": "#579BFC", "position": "0" },
          { "id": "group_1", "title": "In Progress", "color": "#FDAB3D", "position": "1" },
          { "id": "group_2", "title": "Done", "color": "#00C875", "position": "2" }
        ],
        "workspace": {
          "id": "111111",
          "name": "Main workspace"
        },
        "permissions": "everyone",
        "updated_at": "2026-02-09T12:00:00Z"
      }
    ]
  }
}
```

##### Query: Get a single board

```graphql
query GetBoard($boardId: ID!) {
  boards(ids: [$boardId]) {
    id
    name
    description
    columns {
      id
      title
      type
      settings_str
    }
    groups {
      id
      title
      color
    }
    items_page(limit: 50) {
      cursor
      items {
        id
        name
        group {
          id
          title
        }
        column_values {
          id
          text
          value
          type
        }
        created_at
        updated_at
      }
    }
  }
}
```

##### Mutation: Create a board

```graphql
mutation CreateBoard($boardName: String!, $boardKind: BoardKind!, $workspaceId: ID) {
  create_board(board_name: $boardName, board_kind: $boardKind, workspace_id: $workspaceId) {
    id
    name
  }
}
```

**Variables:**

```json
{
  "boardName": "New Project",
  "boardKind": "public",
  "workspaceId": "111111"
}
```

`boardKind` values: `public`, `private`, `share`.

##### Mutation: Archive a board

```graphql
mutation ArchiveBoard($boardId: ID!) {
  archive_board(board_id: $boardId) {
    id
    state
  }
}
```

#### 10.5.2 Board Columns

##### Query: Get columns for a board

```graphql
query GetColumns($boardId: ID!) {
  boards(ids: [$boardId]) {
    columns {
      id
      title
      type
      description
      settings_str
    }
  }
}
```

**Column Types:**

| Type | Description | Value Format |
|---|---|---|
| `name` | Item name (primary) | String |
| `status` | Status/label column | `{"index": 0}` or `{"label": "Done"}` |
| `people` | Person column | `{"personsAndTeams": [{"id": 123, "kind": "person"}]}` |
| `date` | Date column | `{"date": "2026-02-10", "time": "14:00:00"}` |
| `text` | Text column | `"Some text"` |
| `numbers` | Number column | `"42"` |
| `dropdown` | Dropdown column | `{"ids": [1, 2]}` |
| `checkbox` | Checkbox column | `{"checked": "true"}` |
| `timeline` | Timeline (date range) | `{"from": "2026-02-01", "to": "2026-02-15"}` |
| `hour` | Time column | `{"hour": 14, "minute": 30}` |
| `email` | Email column | `{"email": "jane@example.com", "text": "Jane"}` |
| `phone` | Phone column | `{"phone": "+1234567890", "countryShortName": "US"}` |
| `link` | Link column | `{"url": "https://...", "text": "Link text"}` |
| `long_text` | Long text column | `{"text": "Long description..."}` |
| `rating` | Rating column | `{"rating": 4}` |
| `tags` | Tags column | `{"tag_ids": [123, 456]}` |
| `file` | File column | File upload (separate API) |
| `time_tracking` | Time tracking column | See time tracking API |

##### Mutation: Create a column

```graphql
mutation CreateColumn($boardId: ID!, $title: String!, $columnType: ColumnType!) {
  create_column(board_id: $boardId, title: $title, column_type: $columnType) {
    id
    title
    type
  }
}
```

**Variables:**

```json
{
  "boardId": "1234567890",
  "title": "Sprint",
  "columnType": "dropdown"
}
```

#### 10.5.3 Board Groups

##### Query: Get groups

```graphql
query GetGroups($boardId: ID!) {
  boards(ids: [$boardId]) {
    groups {
      id
      title
      color
      position
      items_page(limit: 25) {
        items {
          id
          name
        }
      }
    }
  }
}
```

##### Mutation: Create a group

```graphql
mutation CreateGroup($boardId: ID!, $groupName: String!) {
  create_group(board_id: $boardId, group_name: $groupName) {
    id
    title
  }
}
```

##### Mutation: Delete a group

```graphql
mutation DeleteGroup($boardId: ID!, $groupId: String!) {
  delete_group(board_id: $boardId, group_id: $groupId) {
    id
    deleted
  }
}
```

#### 10.5.4 Board Items (Tasks)

##### Query: Get items

```graphql
query GetItems($boardId: ID!, $limit: Int, $cursor: String) {
  boards(ids: [$boardId]) {
    items_page(limit: $limit, cursor: $cursor) {
      cursor
      items {
        id
        name
        state
        group {
          id
          title
        }
        column_values {
          id
          text
          value
          type
          ... on StatusValue {
            index
            label
            label_style {
              color
            }
          }
          ... on PeopleValue {
            persons_and_teams {
              id
              kind
            }
          }
          ... on DateValue {
            date
            time
          }
        }
        subitems {
          id
          name
          column_values {
            id
            text
            value
          }
        }
        updates {
          id
          body
          text_body
          created_at
          creator {
            id
            name
          }
        }
        created_at
        updated_at
      }
    }
  }
}
```

**Variables:**

```json
{
  "boardId": "1234567890",
  "limit": 25,
  "cursor": null
}
```

##### Query: Get items by column value

```graphql
query GetItemsByColumnValue($boardId: ID!, $columnId: String!, $columnValue: CompareValue!) {
  items_page_by_column_values(
    board_id: $boardId,
    limit: 50,
    columns: [{ column_id: $columnId, column_values: [$columnValue] }]
  ) {
    cursor
    items {
      id
      name
      column_values {
        id
        text
        value
      }
    }
  }
}
```

**Variables:**

```json
{
  "boardId": "1234567890",
  "columnId": "status",
  "columnValue": "Working on it"
}
```

##### Query: Get a specific item

```graphql
query GetItem($itemId: ID!) {
  items(ids: [$itemId]) {
    id
    name
    state
    board {
      id
      name
    }
    group {
      id
      title
    }
    column_values {
      id
      text
      value
      type
    }
    subitems {
      id
      name
    }
    updates(limit: 10) {
      id
      body
      text_body
      created_at
      creator {
        id
        name
      }
    }
    created_at
    updated_at
  }
}
```

##### Mutation: Create an item

```graphql
mutation CreateItem($boardId: ID!, $groupId: String!, $itemName: String!, $columnValues: JSON) {
  create_item(
    board_id: $boardId,
    group_id: $groupId,
    item_name: $itemName,
    column_values: $columnValues
  ) {
    id
    name
    column_values {
      id
      text
      value
    }
  }
}
```

**Variables:**

```json
{
  "boardId": "1234567890",
  "groupId": "new_group",
  "itemName": "Fix login bug",
  "columnValues": "{\"status\": {\"label\": \"Working on it\"}, \"person\": {\"personsAndTeams\": [{\"id\": 9876543, \"kind\": \"person\"}]}, \"date4\": {\"date\": \"2026-02-15\"}, \"text0\": \"Users report being logged out after 5 minutes\", \"priority\": {\"label\": \"High\"}}"
}
```

**Important:** The `columnValues` parameter is a JSON-encoded string (not a nested object). This is a monday.com API quirk that Esmer must handle by `JSON.stringify`-ing the column values object.

##### Mutation: Change column values for an item

```graphql
mutation ChangeColumnValues($boardId: ID!, $itemId: ID!, $columnValues: JSON!) {
  change_multiple_column_values(
    board_id: $boardId,
    item_id: $itemId,
    column_values: $columnValues
  ) {
    id
    name
    column_values {
      id
      text
      value
    }
  }
}
```

**Variables:**

```json
{
  "boardId": "1234567890",
  "itemId": "9999999",
  "columnValues": "{\"status\": {\"label\": \"Done\"}, \"date4\": {\"date\": \"2026-02-10\"}}"
}
```

##### Mutation: Change a single column value

```graphql
mutation ChangeSingleColumnValue($boardId: ID!, $itemId: ID!, $columnId: String!, $value: JSON!) {
  change_simple_column_value(
    board_id: $boardId,
    item_id: $itemId,
    column_id: $columnId,
    value: $value
  ) {
    id
    name
  }
}
```

**Variables:**

```json
{
  "boardId": "1234567890",
  "itemId": "9999999",
  "columnId": "status",
  "value": "{\"label\": \"Done\"}"
}
```

##### Mutation: Move item to group

```graphql
mutation MoveItemToGroup($itemId: ID!, $groupId: String!) {
  move_item_to_group(item_id: $itemId, group_id: $groupId) {
    id
    group {
      id
      title
    }
  }
}
```

##### Mutation: Delete an item

```graphql
mutation DeleteItem($itemId: ID!) {
  delete_item(item_id: $itemId) {
    id
  }
}
```

##### Mutation: Add an update (comment) to an item

```graphql
mutation CreateUpdate($itemId: ID!, $body: String!) {
  create_update(item_id: $itemId, body: $body) {
    id
    body
    text_body
    created_at
    creator {
      id
      name
    }
  }
}
```

**Variables:**

```json
{
  "itemId": "9999999",
  "body": "<p>Delegated by Esmer. Working on this now.</p>"
}
```

**Note:** Update body supports HTML formatting.

#### 10.5.5 Subitems

##### Query: Get subitems

```graphql
query GetSubitems($itemId: [ID!]) {
  items(ids: $itemId) {
    subitems {
      id
      name
      column_values {
        id
        text
        value
      }
      created_at
      updated_at
    }
  }
}
```

##### Mutation: Create a subitem

```graphql
mutation CreateSubitem($parentItemId: ID!, $itemName: String!, $columnValues: JSON) {
  create_subitem(
    parent_item_id: $parentItemId,
    item_name: $itemName,
    column_values: $columnValues
  ) {
    id
    name
    board {
      id
    }
  }
}
```

#### 10.5.6 Users

##### Query: Get users

```graphql
query GetUsers {
  users {
    id
    name
    email
    photo_thumb
    title
    account {
      id
      name
    }
    is_admin
    is_guest
    created_at
  }
}
```

##### Query: Get current user

```graphql
query GetMe {
  me {
    id
    name
    email
    photo_thumb
    account {
      id
      name
      plan {
        max_users
        period
        tier
      }
    }
    teams {
      id
      name
    }
  }
}
```

#### 10.5.7 Workspaces

##### Query: Get workspaces

```graphql
query GetWorkspaces {
  workspaces {
    id
    name
    kind
    description
  }
}
```

### 10.6 Webhooks / Real-time

monday.com supports webhooks for real-time notifications.

**Create a webhook:**

```graphql
mutation CreateWebhook($boardId: ID!, $url: String!, $event: WebhookEventType!) {
  create_webhook(board_id: $boardId, url: $url, event: $event) {
    id
    board_id
  }
}
```

**Variables:**

```json
{
  "boardId": "1234567890",
  "url": "https://esmer.example.com/webhooks/monday",
  "event": "change_status_column_value"
}
```

**Supported Webhook Events:**

| Event | Description |
|---|---|
| `change_column_value` | Any column value changed |
| `change_status_column_value` | Status column changed |
| `change_specific_column_value` | Specific column changed (requires config) |
| `create_item` | Item created |
| `create_update` | Update (comment) created |
| `edit_update` | Update edited |
| `delete_update` | Update deleted |
| `create_subitem` | Subitem created |
| `change_subitem_column_value` | Subitem column value changed |
| `delete_subitem` | Subitem deleted |
| `create_subitem_update` | Subitem update created |

**Challenge Verification:** monday.com sends a `challenge` request when creating a webhook. Your endpoint must respond with:

```json
{
  "challenge": "{the_challenge_token_received}"
}
```

**Webhook Payload:**

```json
{
  "event": {
    "userId": 9876543,
    "originalTriggerUuid": null,
    "boardId": 1234567890,
    "groupId": "new_group",
    "pulseId": 9999999,
    "pulseName": "Fix login bug",
    "columnId": "status",
    "columnType": "color",
    "columnTitle": "Status",
    "value": {
      "label": {
        "index": 1,
        "text": "Done",
        "style": { "color": "#00C875" }
      }
    },
    "previousValue": {
      "label": {
        "index": 0,
        "text": "Working on it",
        "style": { "color": "#FDAB3D" }
      }
    },
    "changedAt": 1707465600.0,
    "isTopGroup": false,
    "triggerTime": "2026-02-09T12:00:00.000Z",
    "subscriptionId": 12345,
    "triggerUuid": "uuid-string"
  }
}
```

### 10.7 Error Handling

monday.com returns errors within the GraphQL response body (HTTP 200):

```json
{
  "errors": [
    {
      "message": "Not Authenticated",
      "locations": [{ "line": 2, "column": 3 }],
      "path": ["boards"],
      "extensions": {
        "code": "NOT_AUTHENTICATED",
        "status_code": 401
      }
    }
  ],
  "data": null,
  "account_id": null
}
```

| Error Code | Description | Esmer Action |
|---|---|---|
| `NOT_AUTHENTICATED` | Invalid or missing token | Re-authenticate |
| `NOT_AUTHORIZED` | Insufficient permissions | Check board/workspace access |
| `RESOURCE_NOT_FOUND` | Board, item, or column not found | Verify IDs |
| `INVALID_ARGUMENT` | Invalid input value | Validate column value formats |
| `COMPLEXITY_BUDGET_EXHAUSTED` | Rate limit exceeded | Wait per `reset_in_x_seconds`; simplify queries |
| `INTERNAL_SERVER_ERROR` | monday.com server error | Retry with exponential backoff |
| `COLUMN_VALUE_ERROR` | Invalid column value format | Check column type and value format |
| `ITEM_LIMIT_EXCEEDED` | Board item limit reached | Inform user; archive old items |
| `CREATE_ITEMS_LIMIT_EXCEEDED` | Too many items created in burst | Slow down item creation |

### 10.8 Mobile-Specific Notes

- **Column Value Formatting:** Each column type has a specific JSON value format. Esmer must fetch the board schema (columns and their `settings_str`) before creating or updating items, then format values accordingly.
- **JSON String Values:** The `column_values` parameter in mutations must be a JSON-encoded string, not a raw object. This is a common source of errors: `JSON.stringify({status: {label: "Done"}})`.
- **Complexity Management:** Monitor the `complexity` field in responses. Complex queries (many columns, nested subitems, updates) consume more budget. Simplify queries for mobile by requesting fewer fields.
- **Cursor Pagination:** monday.com uses cursor-based pagination for items. Store cursors for smooth infinite scroll on mobile.
- **HTML Updates:** Item updates (comments) use HTML formatting. For mobile, Esmer can use simple `<p>` tags for plain text or render HTML for reading.
- **Deep Linking:** `https://{account}.monday.com/boards/{boardId}/pulses/{itemId}` opens an item in the browser. The monday.com mobile app handles universal links.
- **Status Labels vs Indices:** Status columns can be set by label name (`{"label": "Done"}`) or index (`{"index": 1}`). Using label names is more human-readable and robust across board configurations.
- **Board as Schema:** Each monday.com board is essentially a unique schema. Esmer must be adaptable to different board structures per user.

---

*End of Category 03: Tasks & Project Management specification.*
