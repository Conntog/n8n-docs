# Esmer (ESMO) API Integration Specification: File Storage & Cloud Drives

> **Category:** 05 - File Storage & Cloud Drives
> **Version:** 1.0.0
> **Last Updated:** 2026-02-09
> **Status:** Draft

---

## Table of Contents

1. [Category Overview](#category-overview)
2. [Google Drive (API v3)](#1-google-drive-api-v3)
3. [Microsoft OneDrive (Microsoft Graph API)](#2-microsoft-onedrive-microsoft-graph-api)
4. [Microsoft SharePoint (Microsoft Graph API)](#3-microsoft-sharepoint-microsoft-graph-api)
5. [Dropbox (API v2)](#4-dropbox-api-v2)
6. [Box (Box Platform API)](#5-box-box-platform-api)
7. [Nextcloud (WebDAV / OCS API)](#6-nextcloud-webdav--ocs-api)
8. [AWS S3 (REST API)](#7-aws-s3-rest-api)
9. [Google Cloud Storage (JSON API v1)](#8-google-cloud-storage-json-api-v1)
10. [Cross-Service Comparison Matrix](#cross-service-comparison-matrix)

---

## Category Overview

File Storage & Cloud Drives form a critical infrastructure layer for Esmer. As a mobile-first AI delegation assistant, Esmer must seamlessly interact with users' cloud storage to:

- **Attach files** to tasks and conversations (pulling from any connected drive)
- **Save AI-generated artifacts** (reports, summaries, drafted documents) directly to the user's preferred cloud storage
- **Share files** on behalf of the user with teammates or external collaborators
- **Search and retrieve** documents by name, content, or metadata to provide context-aware assistance
- **Organize files** into project-specific folders as part of task delegation workflows
- **Monitor changes** via webhooks to trigger downstream workflows when files are added or modified
- **Generate previews and thumbnails** for mobile display without downloading full files

The services in this category range from consumer-facing drives (Google Drive, OneDrive, Dropbox) to enterprise platforms (SharePoint, Box) to self-hosted solutions (Nextcloud) to infrastructure-grade object storage (AWS S3, Google Cloud Storage).

---

## 1. Google Drive (API v3)

### 1.1 Service Overview

Google Drive is Google's cloud file storage service with deep integration into Google Workspace (Docs, Sheets, Slides). For Esmer, Google Drive serves as the primary storage backend for users in the Google ecosystem. Key capabilities include storing file attachments from delegated tasks, saving AI-generated documents (optionally converting to native Google Docs format), sharing files with collaborators via permission grants, and searching across a user's entire drive using full-text content search.

### 1.2 Authentication

**Method:** OAuth 2.0 (Authorization Code flow)

| Parameter | Value |
|---|---|
| Authorization URL | `https://accounts.google.com/o/oauth2/v2/auth` |
| Token URL | `https://oauth2.googleapis.com/token` |
| Revoke URL | `https://oauth2.googleapis.com/revoke` |
| Grant Type | `authorization_code` |

**Required Scopes:**

| Scope | Purpose |
|---|---|
| `https://www.googleapis.com/auth/drive` | Full read/write access to all Drive files |
| `https://www.googleapis.com/auth/drive.file` | Access only to files created/opened by the app |
| `https://www.googleapis.com/auth/drive.readonly` | Read-only access to file metadata and content |
| `https://www.googleapis.com/auth/drive.metadata.readonly` | Read-only access to file metadata |
| `https://www.googleapis.com/auth/drive.appdata` | Access to the app-specific hidden folder |
| `https://www.googleapis.com/auth/drive.photos.readonly` | Read-only access to photos in Google Photos |

**Recommended for Esmer:** Use `https://www.googleapis.com/auth/drive.file` as the minimum scope (principle of least privilege). Escalate to `https://www.googleapis.com/auth/drive` only when the user explicitly grants full-drive search access.

**Service Account Alternative:** For server-to-server operations, use a Google Service Account with domain-wide delegation. Authenticate using a signed JWT exchanged for an access token at `https://oauth2.googleapis.com/token` with `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer`.

### 1.3 Base URL

```
https://www.googleapis.com/drive/v3
```

Upload endpoint:
```
https://www.googleapis.com/upload/drive/v3
```

### 1.4 Rate Limits

| Limit Type | Value |
|---|---|
| Queries per day | 1,000,000,000 (shared across project) |
| Queries per 100 seconds per user | 12,000 |
| Queries per 100 seconds per project | 12,000 |
| Uploads per day | Varies (generally uncapped; large uploads throttled) |
| File size limit (upload) | 5 TB |
| File size limit (simple upload) | 5 MB |
| File size limit (multipart upload) | 5 MB |
| File size limit (resumable upload) | 5 TB |

**Retry Strategy:** On `403 userRateLimitExceeded` or `429 Too Many Requests`, use exponential backoff starting at 1 second, doubling on each retry, with a maximum of 5 retries. Include jitter (random 0-1s) to avoid thundering herd.

### 1.5 Endpoints

#### File Operations

**Upload a File**
```
POST /upload/drive/v3/files
```
- **Description:** Upload file content and metadata. Supports three upload types.
- **Query Parameters:**
  - `uploadType=media` (simple upload, body is file content, max 5 MB)
  - `uploadType=multipart` (metadata + file in multipart body, max 5 MB)
  - `uploadType=resumable` (initiate resumable session, then upload in chunks, up to 5 TB)
- **Headers (resumable):**
  - `X-Upload-Content-Type: {mime_type}`
  - `X-Upload-Content-Length: {total_bytes}`
  - `Content-Type: application/json; charset=UTF-8`
- **Request Body (multipart):**
  ```
  --boundary
  Content-Type: application/json; charset=UTF-8

  {"name": "file.pdf", "parents": ["folderId"]}
  --boundary
  Content-Type: application/pdf

  {binary file data}
  --boundary--
  ```
- **Response:** `200 OK` with `File` resource (id, name, mimeType, size, webViewLink, etc.)

**Download a File**
```
GET /drive/v3/files/{fileId}?alt=media
```
- **Description:** Download binary file content.
- **Query Parameters:**
  - `alt=media` (required for binary content download)
  - `acknowledgeAbuse=true` (optional, to download flagged files)
- **Response:** Binary stream with `Content-Type` header matching the file's MIME type.

**Export a Google Workspace File**
```
GET /drive/v3/files/{fileId}/export
```
- **Description:** Export Google Docs/Sheets/Slides to a standard format.
- **Query Parameters:**
  - `mimeType` (required): Target format, e.g., `application/pdf`, `text/plain`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document`
- **Response:** Binary stream of the exported file. Maximum export size: 10 MB.

**Get File Metadata**
```
GET /drive/v3/files/{fileId}
```
- **Description:** Retrieve metadata for a file (without downloading content).
- **Query Parameters:**
  - `fields` (optional): Comma-separated list of fields, e.g., `id,name,mimeType,size,modifiedTime,thumbnailLink,webViewLink,permissions`
  - `supportsAllDrives=true` (for shared drive files)
- **Response:** `File` resource JSON.

**List Files**
```
GET /drive/v3/files
```
- **Description:** List files in the user's drive or a specific folder.
- **Query Parameters:**
  - `q` (optional): Search query string, e.g., `'folderId' in parents and trashed=false`
  - `fields` (optional): e.g., `files(id,name,mimeType,size,modifiedTime)`
  - `pageSize` (optional): 1-1000, default 100
  - `pageToken` (optional): Token for pagination
  - `orderBy` (optional): e.g., `modifiedTime desc`
  - `spaces` (optional): `drive`, `appDataFolder`, or `photos`
  - `supportsAllDrives=true`
  - `includeItemsFromAllDrives=true`
  - `corpora` (optional): `user`, `drive`, `domain`, `allDrives`
  - `driveId` (optional): ID of the shared drive to search
- **Response:** `{ files: [File], nextPageToken: string }`

**Search Files**
```
GET /drive/v3/files
```
- **Description:** Search using Google Drive query syntax.
- **Query Examples:**
  - `name contains 'report'`
  - `mimeType = 'application/pdf'`
  - `fullText contains 'quarterly revenue'`
  - `modifiedTime > '2025-01-01T00:00:00'`
  - `'folderId' in parents`
  - `sharedWithMe = true`
- **Response:** Same as List Files.

**Copy a File**
```
POST /drive/v3/files/{fileId}/copy
```
- **Description:** Create a copy of a file.
- **Request Body:**
  ```json
  {
    "name": "Copy of Document",
    "parents": ["destinationFolderId"]
  }
  ```
- **Response:** `File` resource of the new copy.

**Move a File**
```
PATCH /drive/v3/files/{fileId}
```
- **Description:** Move a file by updating its parents.
- **Query Parameters:**
  - `addParents={newFolderId}`
  - `removeParents={oldFolderId}`
- **Response:** Updated `File` resource.

**Update File Metadata**
```
PATCH /drive/v3/files/{fileId}
```
- **Description:** Update file name, description, properties, or content.
- **Request Body:**
  ```json
  {
    "name": "Updated Name.pdf",
    "description": "Updated description",
    "appProperties": { "esmerTaskId": "abc123" }
  }
  ```
- **Response:** Updated `File` resource.

**Update File Content**
```
PATCH /upload/drive/v3/files/{fileId}
```
- **Description:** Replace file content (uses same upload types as create).
- **Query Parameters:** `uploadType=media|multipart|resumable`
- **Response:** Updated `File` resource.

**Delete a File**
```
DELETE /drive/v3/files/{fileId}
```
- **Description:** Permanently delete a file (bypasses trash).
- **Response:** `204 No Content` on success.

**Trash a File**
```
PATCH /drive/v3/files/{fileId}
```
- **Description:** Move to trash.
- **Request Body:** `{ "trashed": true }`
- **Response:** Updated `File` resource.

**Share a File (Create Permission)**
```
POST /drive/v3/files/{fileId}/permissions
```
- **Description:** Grant access to a file.
- **Query Parameters:**
  - `sendNotificationEmail=true|false`
  - `emailMessage=string`
  - `transferOwnership=true|false`
- **Request Body:**
  ```json
  {
    "role": "reader|writer|commenter|owner|organizer|fileOrganizer",
    "type": "user|group|domain|anyone",
    "emailAddress": "user@example.com"
  }
  ```
- **Response:** `Permission` resource.

#### Folder Operations

**Create a Folder**
```
POST /drive/v3/files
```
- **Description:** Create a new folder.
- **Request Body:**
  ```json
  {
    "name": "Project Folder",
    "mimeType": "application/vnd.google-apps.folder",
    "parents": ["parentFolderId"]
  }
  ```
- **Response:** `File` resource with folder metadata.

**Delete a Folder**
```
DELETE /drive/v3/files/{folderId}
```
- **Description:** Permanently delete a folder and its contents.
- **Response:** `204 No Content`.

#### Shared Drive Operations

**Create a Shared Drive**
```
POST /drive/v3/drives
```
- **Query Parameters:** `requestId={unique-idempotency-key}`
- **Request Body:** `{ "name": "Team Drive" }`
- **Response:** `Drive` resource.

**List Shared Drives**
```
GET /drive/v3/drives
```
- **Query Parameters:** `pageSize`, `pageToken`, `q` (search query)
- **Response:** `{ drives: [Drive], nextPageToken: string }`

**Get Shared Drive**
```
GET /drive/v3/drives/{driveId}
```
- **Response:** `Drive` resource.

**Update Shared Drive**
```
PATCH /drive/v3/drives/{driveId}
```
- **Request Body:** `{ "name": "Renamed Drive", "restrictions": {...} }`
- **Response:** Updated `Drive` resource.

**Delete Shared Drive**
```
DELETE /drive/v3/drives/{driveId}
```
- **Response:** `204 No Content`. Only works if the drive is empty.

### 1.6 Webhooks / Real-time Notifications

**Watch for File Changes**
```
POST /drive/v3/files/{fileId}/watch
```
- **Request Body:**
  ```json
  {
    "id": "unique-channel-id",
    "type": "web_hook",
    "address": "https://esmer.app/webhooks/gdrive",
    "expiration": 1735689600000,
    "token": "optional-verification-token"
  }
  ```
- **Response:** `Channel` resource with `resourceId` and `expiration`.
- **Notification Headers:** `X-Goog-Channel-ID`, `X-Goog-Resource-ID`, `X-Goog-Resource-State` (sync, update, add, remove, trash, untrash, change), `X-Goog-Message-Number`.
- **Expiration:** Maximum 24 hours. Must renew via a new `watch` call.

**Watch for Drive Changes**
```
POST /drive/v3/changes/watch
```
- **Query Parameters:** `pageToken` (required, from `changes/startPageToken`)
- **Request Body:** Same as file watch.
- **Description:** Notifies when any file in the drive changes.

**Stop a Watch Channel**
```
POST /drive/v3/channels/stop
```
- **Request Body:** `{ "id": "channel-id", "resourceId": "resource-id" }`
- **Response:** `204 No Content`.

### 1.7 Error Handling

| HTTP Code | Error | Description | Retry? |
|---|---|---|---|
| 400 | `badRequest` | Malformed request | No |
| 401 | `authError` | Invalid or expired token | Refresh token, then retry once |
| 403 | `userRateLimitExceeded` | Per-user rate limit exceeded | Yes, exponential backoff |
| 403 | `rateLimitExceeded` | Project rate limit exceeded | Yes, exponential backoff |
| 403 | `dailyLimitExceeded` | Daily quota exceeded | No, wait until quota resets |
| 403 | `insufficientPermissions` | Missing scope or file access | No |
| 404 | `notFound` | File or drive not found | No |
| 409 | `conflict` | Conflict (e.g., file already exists) | No |
| 416 | `requestedRangeNotSatisfiable` | Invalid byte range in resumable upload | Retry with corrected range |
| 429 | `tooManyRequests` | Rate limit exceeded | Yes, exponential backoff |
| 500 | `internalError` | Google server error | Yes, exponential backoff |
| 502 | `badGateway` | Google server error | Yes, exponential backoff |
| 503 | `serviceUnavailable` | Google server temporarily unavailable | Yes, exponential backoff |

### 1.8 Upload / Download Details

**Simple Upload (`uploadType=media`):**
- Maximum file size: 5 MB
- Single request, body is raw file content
- Set `Content-Type` to file MIME type

**Multipart Upload (`uploadType=multipart`):**
- Maximum file size: 5 MB
- Two-part body: JSON metadata + binary content
- Uses `multipart/related` content type with boundary

**Resumable Upload (`uploadType=resumable`):**
1. **Initiate:** `POST /upload/drive/v3/files?uploadType=resumable` with metadata. Returns `200 OK` with `Location` header containing the upload URI.
2. **Upload chunks:** `PUT {upload_uri}` with `Content-Range: bytes {start}-{end}/{total}` header. Chunk size must be a multiple of 256 KB (262144 bytes). Minimum recommended chunk size: 8 MB.
3. **Resume after interruption:** `PUT {upload_uri}` with `Content-Range: bytes */{total}` to query upload status. Response `308 Resume Incomplete` with `Range` header indicating bytes received.
4. **Upload URI validity:** 1 week from creation.

**Download:**
- Use `alt=media` query parameter for binary download
- For Google Workspace files, use the `/export` endpoint with target `mimeType`
- Supports partial download with `Range` HTTP header
- No inherent file size limit on download

### 1.9 Mobile-Specific Notes

- **Thumbnails:** The `thumbnailLink` field in file metadata provides a short-lived URL to a thumbnail image. Append `=s{size}` to control dimensions (e.g., `=s220` for 220px). Valid for approximately 8 hours.
- **File Previews:** Use `webViewLink` to open the file in Google's web viewer (works in mobile WebView). Use `webContentLink` for direct download links.
- **Share Links:** Generate with the Permissions endpoint, setting `type=anyone` and `role=reader` for public links. The `webViewLink` in the response serves as the shareable URL.
- **Offline Consideration:** Cache file metadata (id, name, mimeType, thumbnailLink) locally. Queue upload/download operations when offline and execute when connectivity returns.
- **Small Screen Optimization:** Request only essential fields via the `fields` parameter to minimize payload size.

---

## 2. Microsoft OneDrive (Microsoft Graph API)

### 2.1 Service Overview

Microsoft OneDrive is Microsoft's cloud storage service integrated into the Microsoft 365 ecosystem. For Esmer, OneDrive provides file storage for users in Microsoft-centric organizations. Key uses include storing and retrieving task attachments, saving AI-generated content (Word docs, Excel files), sharing files via link or direct permission, and leveraging Microsoft's full-text search across OneDrive contents. OneDrive supports personal accounts (MSA) and work/school accounts (Azure AD).

### 2.2 Authentication

**Method:** OAuth 2.0 (Authorization Code flow via Microsoft Identity Platform v2.0)

| Parameter | Value |
|---|---|
| Authorization URL | `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize` |
| Token URL | `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token` |
| Grant Type | `authorization_code` |
| Tenant Values | `common` (personal + work), `organizations` (work/school only), `consumers` (personal only), or specific tenant ID |

**Government Cloud Endpoints:**

| Cloud | Authorization Base |
|---|---|
| US Government | `https://login.microsoftonline.us` |
| US Government DOD | `https://login.microsoftonline.us` |
| China (21Vianet) | `https://login.chinacloudapi.cn` |

**Required Scopes (Delegated):**

| Scope | Purpose |
|---|---|
| `Files.Read` | Read user's files |
| `Files.Read.All` | Read all files user can access |
| `Files.ReadWrite` | Read/write user's files |
| `Files.ReadWrite.All` | Read/write all files user can access |
| `Sites.Read.All` | Read items in all site collections (for SharePoint-backed drives) |
| `Sites.ReadWrite.All` | Read/write items in all site collections |
| `offline_access` | Obtain refresh tokens |
| `User.Read` | Sign in and read user profile |

**Recommended for Esmer:** `Files.ReadWrite.All offline_access User.Read`

### 2.3 Base URL

```
https://graph.microsoft.com/v1.0
```

Government Cloud Base URLs:

| Cloud | Base URL |
|---|---|
| US Government L4 | `https://graph.microsoft.us/v1.0` |
| US Government L5 (DOD) | `https://dod-graph.microsoft.us/v1.0` |
| China (21Vianet) | `https://microsoftgraph.chinacloudapi.cn/v1.0` |

### 2.4 Rate Limits

| Limit Type | Value |
|---|---|
| Per app per tenant | 10,000 requests per 10 minutes |
| Per app across all tenants | 100,000 requests per 10 minutes (approx.) |
| Per user per app | Varies; typically throttled at ~60 requests/minute for drive operations |
| Concurrent uploads per user | 4 simultaneous upload sessions |
| Upload session expiration | 48 hours of inactivity |

**Throttling Response:** `429 Too Many Requests` with `Retry-After` header (value in seconds). Always respect the `Retry-After` value.

### 2.5 Endpoints

All paths are relative to `https://graph.microsoft.com/v1.0`.

#### File Operations

**Upload a Small File (up to 4 MB)**
```
PUT /me/drive/items/{parent-item-id}:/{filename}:/content
```
or
```
PUT /me/drive/root:/{path/to/filename}:/content
```
- **Description:** Upload file content directly. Maximum 4 MB.
- **Headers:** `Content-Type: {mime_type}`
- **Request Body:** Raw binary file content.
- **Response:** `DriveItem` resource.

**Create Upload Session (for files > 4 MB, up to 250 GB)**
```
POST /me/drive/items/{parent-item-id}:/{filename}:/createUploadSession
```
- **Description:** Initiate a resumable upload session.
- **Request Body:**
  ```json
  {
    "item": {
      "@microsoft.graph.conflictBehavior": "rename|replace|fail",
      "name": "largefile.zip"
    }
  }
  ```
- **Response:** `{ "uploadUrl": "https://...", "expirationDateTime": "..." }`

**Upload Bytes to Session**
```
PUT {uploadUrl}
```
- **Headers:**
  - `Content-Length: {chunk-size}`
  - `Content-Range: bytes {start}-{end}/{total}`
- **Chunk Size:** Must be a multiple of 320 KiB (327,680 bytes). Recommended: 5-10 MB. Maximum: 60 MiB per request.
- **Response:** `202 Accepted` (in progress) or `201 Created`/`200 OK` (complete) with `DriveItem`.

**Download a File**
```
GET /me/drive/items/{item-id}/content
```
- **Description:** Download file content. Returns `302` redirect to a pre-authenticated download URL.
- **Response:** `302 Found` with `Location` header pointing to the download URL.

**Get File Metadata**
```
GET /me/drive/items/{item-id}
```
- **Query Parameters:**
  - `$select` (optional): e.g., `id,name,size,lastModifiedDateTime,webUrl,@microsoft.graph.downloadUrl`
  - `$expand` (optional): e.g., `thumbnails`
- **Response:** `DriveItem` resource.

**List Children of a Folder**
```
GET /me/drive/items/{item-id}/children
```
or
```
GET /me/drive/root/children
```
- **Query Parameters:**
  - `$top` (page size, max 200)
  - `$skipToken` (pagination)
  - `$orderby` (e.g., `name asc`, `lastModifiedDateTime desc`)
  - `$select` (field selection)
  - `$filter` (OData filter)
- **Response:** `{ "value": [DriveItem], "@odata.nextLink": "..." }`

**Search Files**
```
GET /me/drive/root/search(q='{search-text}')
```
- **Description:** Full-text search across file names and content.
- **Query Parameters:** `$top`, `$select`, `$filter`
- **Response:** `{ "value": [DriveItem] }` with `@odata.nextLink` for pagination.

**Copy a File**
```
POST /me/drive/items/{item-id}/copy
```
- **Description:** Asynchronous copy operation.
- **Request Body:**
  ```json
  {
    "parentReference": { "driveId": "...", "id": "..." },
    "name": "Copy of file.docx"
  }
  ```
- **Response:** `202 Accepted` with `Location` header pointing to a monitor URL. Poll the monitor URL to check completion.

**Move / Rename a File**
```
PATCH /me/drive/items/{item-id}
```
- **Request Body:**
  ```json
  {
    "parentReference": { "id": "new-parent-folder-id" },
    "name": "new-name.pdf"
  }
  ```
- **Response:** Updated `DriveItem`.

**Delete a File**
```
DELETE /me/drive/items/{item-id}
```
- **Description:** Move to recycle bin.
- **Response:** `204 No Content`.

**Create a Sharing Link**
```
POST /me/drive/items/{item-id}/createLink
```
- **Request Body:**
  ```json
  {
    "type": "view|edit|embed",
    "scope": "anonymous|organization",
    "password": "optional-password",
    "expirationDateTime": "2026-12-31T00:00:00Z"
  }
  ```
- **Response:** `Permission` resource with `link.webUrl`.

**Invite / Share with Specific Users**
```
POST /me/drive/items/{item-id}/invite
```
- **Request Body:**
  ```json
  {
    "requireSignIn": true,
    "sendInvitation": true,
    "roles": ["read|write"],
    "recipients": [{ "email": "user@example.com" }],
    "message": "Here's the file you requested"
  }
  ```
- **Response:** Array of `Permission` resources.

#### Folder Operations

**Create a Folder**
```
POST /me/drive/items/{parent-item-id}/children
```
- **Request Body:**
  ```json
  {
    "name": "New Folder",
    "folder": {},
    "@microsoft.graph.conflictBehavior": "rename|replace|fail"
  }
  ```
- **Response:** `DriveItem` resource.

**Delete a Folder**
```
DELETE /me/drive/items/{item-id}
```
- **Response:** `204 No Content`.

**Get Folder (with children)**
```
GET /me/drive/items/{item-id}?$expand=children
```
- **Response:** `DriveItem` with embedded `children` array.

### 2.6 Webhooks / Real-time Notifications

**Create Subscription**
```
POST /subscriptions
```
- **Request Body:**
  ```json
  {
    "changeType": "updated",
    "notificationUrl": "https://esmer.app/webhooks/onedrive",
    "resource": "/me/drive/root",
    "expirationDateTime": "2026-02-16T00:00:00Z",
    "clientState": "secret-verification-string"
  }
  ```
- **Maximum Subscription Duration:** 43200 minutes (30 days) for drive items.
- **Validation:** Microsoft sends a `POST` with `validationToken` query parameter to your `notificationUrl`. You must respond with `200 OK` and the token as plain text body within 10 seconds.
- **Notification Payload:** Contains `resource`, `changeType`, `clientState`, and `resourceData`.

**Renew Subscription**
```
PATCH /subscriptions/{subscription-id}
```
- **Request Body:** `{ "expirationDateTime": "2026-03-16T00:00:00Z" }`

**Delete Subscription**
```
DELETE /subscriptions/{subscription-id}
```

**Delta Queries (Alternative to Webhooks)**
```
GET /me/drive/root/delta
```
- **Description:** Track incremental changes since last sync.
- **Query Parameters:** `$select`, `$top`, `token` (from `@odata.deltaLink`)
- **Response:** `{ "value": [DriveItem], "@odata.deltaLink": "..." }` or `@odata.nextLink` for pagination.

### 2.7 Error Handling

| HTTP Code | Error Code | Description | Retry? |
|---|---|---|---|
| 400 | `invalidRequest` | Malformed request | No |
| 401 | `unauthenticated` | Token expired or invalid | Refresh token, retry once |
| 403 | `accessDenied` | Insufficient permissions | No |
| 404 | `itemNotFound` | File/folder not found | No |
| 409 | `nameAlreadyExists` | Conflict on create | No (use conflictBehavior) |
| 412 | `preconditionFailed` | ETag mismatch | Retry with fresh ETag |
| 429 | `activityLimitReached` | Rate limit exceeded | Yes, respect `Retry-After` header |
| 500 | `generalException` | Server error | Yes, exponential backoff |
| 502 | `badGateway` | Server error | Yes, exponential backoff |
| 503 | `serviceNotAvailable` | Service temporarily unavailable | Yes, respect `Retry-After` header |
| 507 | `notAllowed` | Quota exceeded | No |

### 2.8 Upload / Download Details

**Simple Upload (PUT):**
- Maximum 4 MB
- Single PUT request with raw binary body
- Atomic operation

**Upload Session (Resumable):**
- For files 4 MB to 250 GB
- Chunk size: multiple of 320 KiB (recommended 5-10 MB)
- Maximum chunk size: 60 MiB per PUT
- Session expires after 48 hours of inactivity
- On failure, query upload status: `GET {uploadUrl}` returns `Range` header with received bytes
- Cancel session: `DELETE {uploadUrl}`

**Download:**
- `GET .../content` returns 302 redirect to pre-authenticated URL
- Pre-authenticated URL valid for a short period (minutes)
- Supports `Range` header for partial downloads
- Use `@microsoft.graph.downloadUrl` from metadata for direct download (valid for ~1 hour)

### 2.9 Mobile-Specific Notes

- **Thumbnails:** Request via `GET /me/drive/items/{id}/thumbnails` or expand with `?$expand=thumbnails`. Returns `small` (96x96), `medium` (176x176), and `large` (800x800) thumbnail URLs. Custom sizes: `GET /me/drive/items/{id}/thumbnails/0/{size}` where size is like `c200x200` (crop) or `200x200` (fit).
- **File Previews:** `POST /me/drive/items/{id}/preview` returns an embeddable preview URL that works in mobile WebView.
- **Share Links:** Use `createLink` endpoint with `type=view` and `scope=anonymous` for public links.
- **Conflict Handling:** Use `@microsoft.graph.conflictBehavior` on uploads to handle naming conflicts automatically.
- **Bandwidth Optimization:** Use `$select` to limit response fields. Use thumbnail URLs instead of downloading full images for display.

---

## 3. Microsoft SharePoint (Microsoft Graph API)

### 3.1 Service Overview

Microsoft SharePoint is an enterprise document management and collaboration platform. For Esmer, SharePoint integration enables access to organizational document libraries, team sites, and structured list data. This is essential for enterprise users whose documents live in SharePoint rather than personal OneDrive. Key uses include uploading meeting notes or task artifacts to team document libraries, reading list items (e.g., project trackers, task boards), and managing structured data alongside files.

### 3.2 Authentication

**Method:** OAuth 2.0 (same Microsoft Identity Platform v2.0 as OneDrive)

| Parameter | Value |
|---|---|
| Authorization URL | `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize` |
| Token URL | `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token` |
| Grant Type | `authorization_code` |

**Required Scopes (Delegated):**

| Scope | Purpose |
|---|---|
| `Sites.Read.All` | Read items in all site collections |
| `Sites.ReadWrite.All` | Read/write items in all site collections |
| `Sites.Manage.All` | Create, edit, and delete items and lists |
| `Files.ReadWrite.All` | Full file access (covers drive items in SP sites) |
| `offline_access` | Refresh tokens |

**Application Permissions (daemon/service):**

| Scope | Purpose |
|---|---|
| `Sites.Read.All` | Read all site data without a signed-in user |
| `Sites.ReadWrite.All` | Read/write all site data |
| `Sites.FullControl.All` | Full control of all site collections |

### 3.3 Base URL

```
https://graph.microsoft.com/v1.0
```

SharePoint REST API (alternative, legacy):
```
https://{tenant}.sharepoint.com/_api
```

### 3.4 Rate Limits

SharePoint throttling via Microsoft Graph follows the same patterns as OneDrive, plus additional SharePoint-specific limits:

| Limit Type | Value |
|---|---|
| Per app per tenant (Graph) | 10,000 requests per 10 minutes |
| SharePoint REST API | Throttled dynamically based on tenant health |
| Concurrent operations per site collection | Resource-dependent; heavy operations throttled more aggressively |
| File upload size (simple) | 4 MB via Graph, 250 MB via SP REST |
| File upload size (session) | 250 GB via Graph upload sessions |

**Throttling Headers:** `Retry-After` (seconds), `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`.

### 3.5 Endpoints

All paths relative to `https://graph.microsoft.com/v1.0`.

#### Site Discovery

**Get Site by Path**
```
GET /sites/{hostname}:/{server-relative-path}
```
- Example: `GET /sites/contoso.sharepoint.com:/sites/TeamSite`
- **Response:** `Site` resource with `id`, `name`, `webUrl`.

**Search Sites**
```
GET /sites?search={query}
```
- **Response:** `{ "value": [Site] }`

**List Drives (Document Libraries) in a Site**
```
GET /sites/{site-id}/drives
```
- **Response:** `{ "value": [Drive] }` where each Drive represents a document library.

#### File Operations

**Upload a File (small, up to 4 MB)**
```
PUT /sites/{site-id}/drive/items/{parent-id}:/{filename}:/content
```
- **Headers:** `Content-Type: {mime_type}`
- **Request Body:** Raw binary
- **Response:** `DriveItem` resource.

**Create Upload Session (large files)**
```
POST /sites/{site-id}/drive/items/{parent-id}:/{filename}:/createUploadSession
```
- Same mechanics as OneDrive upload sessions (see Section 2.5).

**Download a File**
```
GET /sites/{site-id}/drive/items/{item-id}/content
```
- **Response:** `302 Found` redirect to download URL.

**Update a File**
```
PUT /sites/{site-id}/drive/items/{item-id}/content
```
- **Headers:** `Content-Type: {mime_type}`
- **Request Body:** New binary content.
- **Response:** Updated `DriveItem`.

#### List Operations

**Get Lists in a Site**
```
GET /sites/{site-id}/lists
```
- **Query Parameters:** `$select`, `$filter`, `$top`
- **Response:** `{ "value": [List] }`

**Get a Specific List**
```
GET /sites/{site-id}/lists/{list-id}
```
- **Response:** `List` resource with `id`, `displayName`, `columns`, etc.

**Get List Items**
```
GET /sites/{site-id}/lists/{list-id}/items
```
- **Query Parameters:**
  - `$expand=fields` (include column values)
  - `$select=fields/Title,fields/Status`
  - `$filter=fields/Status eq 'Active'`
  - `$top`, `$skipToken`
- **Response:** `{ "value": [ListItem] }`

**Create a List Item**
```
POST /sites/{site-id}/lists/{list-id}/items
```
- **Request Body:**
  ```json
  {
    "fields": {
      "Title": "New Task",
      "Status": "In Progress",
      "DueDate": "2026-03-01"
    }
  }
  ```
- **Response:** `ListItem` resource.

**Update a List Item**
```
PATCH /sites/{site-id}/lists/{list-id}/items/{item-id}/fields
```
- **Request Body:** `{ "Status": "Completed" }`
- **Response:** Updated `FieldValueSet`.

**Delete a List Item**
```
DELETE /sites/{site-id}/lists/{list-id}/items/{item-id}
```
- **Response:** `204 No Content`.

### 3.6 Webhooks / Real-time Notifications

SharePoint uses the same Microsoft Graph subscription model as OneDrive.

**Create Subscription for a List**
```
POST /subscriptions
```
- **Request Body:**
  ```json
  {
    "changeType": "created,updated,deleted",
    "notificationUrl": "https://esmer.app/webhooks/sharepoint",
    "resource": "/sites/{site-id}/lists/{list-id}",
    "expirationDateTime": "2026-02-16T00:00:00Z",
    "clientState": "secret-verification-string"
  }
  ```
- **Maximum Subscription Duration:** 43200 minutes (30 days).

**Create Subscription for a Drive (Document Library)**
```
POST /subscriptions
```
- **resource:** `/sites/{site-id}/drive/root`

**Delta Queries for Document Libraries**
```
GET /sites/{site-id}/drive/root/delta
```
- Same pattern as OneDrive delta queries.

### 3.7 Error Handling

Same error codes as OneDrive (Section 2.7), plus:

| HTTP Code | Error Code | Description | Retry? |
|---|---|---|---|
| 403 | `notAllowed` | Operation not allowed on this resource | No |
| 409 | `conflict` | Item version conflict | Retry with updated etag |
| 423 | `locked` | Resource is locked for editing | Yes, retry after delay |
| 429 | `tooManyRequests` | Throttled | Yes, respect `Retry-After` |
| 507 | `insufficientStorage` | Site quota exceeded | No |

### 3.8 Upload / Download Details

Identical to OneDrive mechanics (Section 2.8). SharePoint document libraries are accessed as drives through the Graph API. The same 4 MB simple upload limit and 250 GB resumable upload limit apply.

### 3.9 Mobile-Specific Notes

- **Thumbnails:** Same as OneDrive -- use `?$expand=thumbnails` or `/thumbnails` endpoint on drive items.
- **Site Navigation:** On mobile, present a simplified site picker. Use `GET /sites?search=` to let users find their sites by name.
- **List Views:** SharePoint lists can have many columns. For mobile display, use `$select` on the `fields` expand to request only the columns needed for the current view.
- **Offline Caching:** Cache list item data and file metadata locally. SharePoint list item changes can be tracked via delta queries for efficient sync.

---

## 4. Dropbox (API v2)

### 4.1 Service Overview

Dropbox is a widely used consumer and business cloud storage service known for reliable file syncing. For Esmer, Dropbox integration enables file management for users who rely on Dropbox as their primary storage. Key capabilities include uploading task artifacts, downloading files for AI processing, organizing files into folders, and searching across the user's Dropbox contents.

### 4.2 Authentication

**Method:** OAuth 2.0 (Authorization Code flow)

| Parameter | Value |
|---|---|
| Authorization URL | `https://www.dropbox.com/oauth2/authorize` |
| Token URL | `https://api.dropboxapi.com/oauth2/token` |
| Revoke URL | `https://api.dropboxapi.com/oauth2/token/revoke` |
| Grant Type | `authorization_code` |
| Token Type | Short-lived (4 hours) with refresh token |

**Required Scopes (Granular, PKCE-enabled):**

| Scope | Purpose |
|---|---|
| `files.metadata.read` | Read file/folder metadata |
| `files.metadata.write` | Write file/folder metadata (move, rename, delete) |
| `files.content.read` | Read file content (download) |
| `files.content.write` | Write file content (upload) |
| `sharing.read` | Read sharing info and shared links |
| `sharing.write` | Create/manage shared links and folder sharing |
| `file_requests.read` | Read file requests |
| `file_requests.write` | Create/manage file requests |
| `account_info.read` | Read basic account information |

**Recommended for Esmer:** `files.metadata.read files.metadata.write files.content.read files.content.write sharing.read sharing.write account_info.read`

**App Types:**
- **Scoped access:** Modern approach with granular permissions
- **App folder:** Access limited to `/Apps/{app-name}/` only
- **Full Dropbox:** Access to entire user's Dropbox

### 4.3 Base URL

```
https://api.dropboxapi.com/2
```

Content operations (upload/download):
```
https://content.dropboxapi.com/2
```

### 4.4 Rate Limits

| Limit Type | Value |
|---|---|
| Write operations (per user) | ~Thousands per endpoint per 15-minute window |
| Individual endpoint rate limits | Vary; no published fixed numbers |
| Upload file size (single request) | 150 MB |
| Upload file size (upload session) | 350 GB |
| Batch operations | Max 1000 entries per batch |
| `list_folder/continue` | No separate limit, but paginated at 2000 entries |

**Note:** Dropbox uses dynamic rate limiting. The API returns `429 Too Many Requests` with a `Retry-After` header.

### 4.5 Endpoints

All paths relative to `https://api.dropboxapi.com/2` unless noted. Content endpoints use `https://content.dropboxapi.com/2`.

#### File Operations

**Upload a File (up to 150 MB)**
```
POST https://content.dropboxapi.com/2/files/upload
```
- **Headers:**
  - `Content-Type: application/octet-stream`
  - `Dropbox-API-Arg: {"path": "/folder/file.pdf", "mode": "add|overwrite|update", "autorename": true, "mute": false}`
- **Request Body:** Raw binary file data.
- **Response:** `FileMetadata` (name, path_lower, id, size, content_hash, etc.)

**Start Upload Session (for files > 150 MB)**
```
POST https://content.dropboxapi.com/2/files/upload_session/start
```
- **Headers:**
  - `Content-Type: application/octet-stream`
  - `Dropbox-API-Arg: {"close": false}`
- **Request Body:** First chunk of binary data.
- **Response:** `{ "session_id": "..." }`

**Append to Upload Session**
```
POST https://content.dropboxapi.com/2/files/upload_session/append_v2
```
- **Headers:**
  - `Dropbox-API-Arg: {"cursor": {"session_id": "...", "offset": <byte_offset>}, "close": false}`
- **Request Body:** Next chunk of binary data.
- **Response:** Empty on success.
- **Chunk Size:** Recommended 4-8 MB per chunk. Maximum 150 MB per append.

**Finish Upload Session**
```
POST https://content.dropboxapi.com/2/files/upload_session/finish
```
- **Headers:**
  - `Dropbox-API-Arg: {"cursor": {"session_id": "...", "offset": <final_offset>}, "commit": {"path": "/folder/file.zip", "mode": "add", "autorename": true}}`
- **Request Body:** Final chunk (can be empty if all data already sent).
- **Response:** `FileMetadata`.

**Batch Upload Finish**
```
POST /files/upload_session/finish_batch
```
- **Description:** Commit multiple upload sessions in one call.
- **Request Body:**
  ```json
  {
    "entries": [
      { "cursor": { "session_id": "...", "offset": 0 }, "commit": { "path": "/a.txt" } },
      { "cursor": { "session_id": "...", "offset": 0 }, "commit": { "path": "/b.txt" } }
    ]
  }
  ```
- **Response:** `{ ".tag": "async_job_id", "async_job_id": "..." }` or `{ ".tag": "complete", "entries": [...] }`

**Download a File**
```
POST https://content.dropboxapi.com/2/files/download
```
- **Headers:**
  - `Dropbox-API-Arg: {"path": "/folder/file.pdf"}`
- **Response:** Binary file content. Metadata in `Dropbox-API-Result` response header as JSON.

**Download a Folder as ZIP**
```
POST https://content.dropboxapi.com/2/files/download_zip
```
- **Headers:**
  - `Dropbox-API-Arg: {"path": "/folder"}`
- **Response:** ZIP binary stream. Maximum folder size: 20 GB or 10,000 files.

**Get File Metadata**
```
POST /files/get_metadata
```
- **Request Body:**
  ```json
  {
    "path": "/folder/file.pdf",
    "include_media_info": true,
    "include_deleted": false,
    "include_has_explicit_shared_members": true
  }
  ```
- **Response:** `FileMetadata` or `FolderMetadata`.

**List Folder Contents**
```
POST /files/list_folder
```
- **Request Body:**
  ```json
  {
    "path": "/folder",
    "recursive": false,
    "include_media_info": true,
    "include_deleted": false,
    "limit": 2000
  }
  ```
- **Response:** `{ "entries": [Metadata], "cursor": "...", "has_more": true|false }`

**Continue Listing**
```
POST /files/list_folder/continue
```
- **Request Body:** `{ "cursor": "..." }`
- **Response:** Same structure as `list_folder`.

**Search Files**
```
POST /files/search_v2
```
- **Request Body:**
  ```json
  {
    "query": "quarterly report",
    "options": {
      "path": "/Documents",
      "max_results": 100,
      "file_status": "active",
      "filename_only": false,
      "file_extensions": ["pdf", "docx"],
      "file_categories": ["document"]
    },
    "match_field_options": { "include_highlights": true }
  }
  ```
- **Response:** `{ "matches": [SearchMatch], "has_more": true|false, "cursor": "..." }`

**Copy a File**
```
POST /files/copy_v2
```
- **Request Body:**
  ```json
  {
    "from_path": "/source/file.pdf",
    "to_path": "/dest/file.pdf",
    "autorename": true
  }
  ```
- **Response:** `{ "metadata": FileMetadata }`

**Move a File**
```
POST /files/move_v2
```
- **Request Body:**
  ```json
  {
    "from_path": "/old/file.pdf",
    "to_path": "/new/file.pdf",
    "autorename": true
  }
  ```
- **Response:** `{ "metadata": FileMetadata }`

**Delete a File**
```
POST /files/delete_v2
```
- **Request Body:** `{ "path": "/folder/file.pdf" }`
- **Response:** `{ "metadata": FileMetadata }` (with `.tag` = "deleted")

#### Folder Operations

**Create a Folder**
```
POST /files/create_folder_v2
```
- **Request Body:** `{ "path": "/new-folder", "autorename": false }`
- **Response:** `{ "metadata": FolderMetadata }`

**Copy a Folder**
```
POST /files/copy_v2
```
- Same as file copy; Dropbox handles folder recursion automatically.

**Move a Folder**
```
POST /files/move_v2
```
- Same as file move.

**Delete a Folder**
```
POST /files/delete_v2
```
- Same as file delete; recursively deletes contents.

#### Sharing Operations

**Create a Shared Link**
```
POST /sharing/create_shared_link_with_settings
```
- **Request Body:**
  ```json
  {
    "path": "/folder/file.pdf",
    "settings": {
      "requested_visibility": "public|team_only|password",
      "link_password": "optional",
      "expires": "2026-12-31T00:00:00Z",
      "audience": "public|team|no_one",
      "access": "viewer|editor|max"
    }
  }
  ```
- **Response:** `SharedLinkMetadata` with `url`, `name`, `path_lower`.

**List Shared Links**
```
POST /sharing/list_shared_links
```
- **Request Body:** `{ "path": "/folder/file.pdf", "direct_only": true }`
- **Response:** `{ "links": [SharedLinkMetadata], "has_more": false }`

### 4.6 Webhooks / Real-time Notifications

**Webhook Registration:** Done via the Dropbox App Console (not via API). Configure a webhook URL in the app settings.

**Verification:** Dropbox sends a `GET` request with `challenge` query parameter. Respond with `200 OK` and the challenge value as `text/plain` body, `X-Content-Type-Options: nosniff` header.

**Change Notifications:**
- Dropbox sends a `POST` to your webhook URL with:
  ```json
  {
    "list_folder": {
      "accounts": ["dbid:AAH4f99T0taONIb-OurWxbNQ6ywGRopQngc"]
    },
    "delta": {
      "users": [12345678]
    }
  }
  ```
- This only tells you *which accounts* changed, not *what* changed. You must then call `list_folder/continue` with the saved cursor to get actual changes.
- **Latency:** Notifications may be batched; expect up to ~120 seconds delay.

### 4.7 Error Handling

| HTTP Code | Error Tag | Description | Retry? |
|---|---|---|---|
| 400 | `bad_input` | Malformed request | No |
| 401 | `invalid_access_token` | Token expired/revoked | Refresh token, retry |
| 403 | `access_denied` | App lacks permission | No |
| 409 | `path/conflict` | File already exists (path conflict) | No (use autorename) |
| 409 | `path/not_found` | Path does not exist | No |
| 409 | `path/insufficient_space` | User quota exceeded | No |
| 429 | `too_many_requests` | Rate limited | Yes, respect `Retry-After` |
| 500 | Internal server error | Dropbox server error | Yes, exponential backoff |
| 503 | Service unavailable | Temporary outage | Yes, exponential backoff |

**Dropbox-Specific:** All API errors use the `.tag` discriminated union pattern. The response body contains a structured JSON error:
```json
{
  "error_summary": "path/not_found/...",
  "error": {
    ".tag": "path",
    "path": { ".tag": "not_found" }
  }
}
```

### 4.8 Upload / Download Details

**Simple Upload:**
- Maximum 150 MB
- Single POST with `application/octet-stream` body
- Path and write mode specified in `Dropbox-API-Arg` header as JSON

**Upload Session (Resumable):**
1. `upload_session/start` -- returns session_id
2. `upload_session/append_v2` -- append chunks (recommended 4-8 MB, max 150 MB per append)
3. `upload_session/finish` -- commit with final path
- Maximum total file size: 350 GB
- Sessions expire after 7 days of inactivity
- Batch finish available for committing multiple sessions atomically

**Download:**
- Path specified in `Dropbox-API-Arg` header (not URL)
- File metadata returned in `Dropbox-API-Result` response header
- Supports `Range` header for partial downloads
- `download_zip` for entire folders (max 20 GB / 10,000 files)

**Content Hash:** Dropbox provides a `content_hash` field (SHA-256 based, computed in 4 MB blocks) for verifying upload/download integrity.

### 4.9 Mobile-Specific Notes

- **Thumbnails:** `POST https://content.dropboxapi.com/2/files/get_thumbnail_v2` with `Dropbox-API-Arg` specifying `resource`, `format` (jpeg/png), `size` (w32h32 through w2048h1536), `mode` (strict/bestfit/fitone_bestfit). Supports images, PDFs, and video files.
- **File Previews:** `POST https://content.dropboxapi.com/2/files/get_preview` returns a PDF or HTML preview of document files (Word, Excel, PowerPoint, text, code). Works for non-image files.
- **Share Links:** Use `create_shared_link_with_settings` with `requested_visibility=public` for instant mobile sharing.
- **Content Hash:** Use the `content_hash` to verify downloads on mobile where connections may be unreliable.
- **Bandwidth:** Dropbox header-based API arg pattern means metadata is in headers, reducing body parsing overhead.

---

## 5. Box (Box Platform API)

### 5.1 Service Overview

Box is an enterprise-focused cloud content management platform emphasizing security, compliance, and governance. For Esmer, Box integration serves enterprise users who store regulated or sensitive documents in Box. Key capabilities include secure file upload/download with granular permissions, metadata-driven search, shared link generation with password protection and expiration, and enterprise workflow triggers via webhooks.

### 5.2 Authentication

**Method:** OAuth 2.0 (Authorization Code flow)

| Parameter | Value |
|---|---|
| Authorization URL | `https://account.box.com/api/oauth2/authorize` |
| Token URL | `https://api.box.com/oauth2/token` |
| Revoke URL | `https://api.box.com/oauth2/revoke` |
| Grant Type | `authorization_code` |
| Access Token Lifetime | 60 minutes |
| Refresh Token Lifetime | 60 days (single use; each refresh returns a new refresh token) |

**Alternative: JWT (Server-to-Server)**

For service accounts, Box supports JWT-based authentication:
1. Generate RSA keypair in Box Developer Console
2. Create JWT assertion signed with private key
3. Exchange JWT for access token at `https://api.box.com/oauth2/token` with `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer`
4. Use `box_subject_type=enterprise` or `box_subject_type=user` with `box_subject_id`

**Alternative: Client Credentials Grant**

```
POST https://api.box.com/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id={client_id}
&client_secret={client_secret}
&box_subject_type=enterprise
&box_subject_id={enterprise_id}
```

**Scopes (Downscoping):**

Box uses token downscoping rather than OAuth scopes. Exchange a full token for a limited one:

| Scope | Purpose |
|---|---|
| `base_explorer` | Browse and list content |
| `item_preview` | Preview files |
| `item_download` | Download files |
| `item_upload` | Upload files |
| `item_share` | Share files and folders |
| `item_delete` | Delete files and folders |
| `item_rename` | Rename files and folders |
| `annotation_edit` | Create/edit annotations |
| `annotation_view_all` | View all annotations |
| `annotation_view_self` | View own annotations |

### 5.3 Base URL

```
https://api.box.com/2.0
```

Upload endpoint:
```
https://upload.box.com/api/2.0
```

### 5.4 Rate Limits

| Limit Type | Value |
|---|---|
| API calls per second (per user) | 10 |
| API calls per second (per app per enterprise) | 1,000 |
| Upload file size (direct) | 50 MB |
| Upload file size (chunked) | 50 GB |
| Search queries | Subject to indexing delay (up to 10 minutes for new content) |

**Throttling Response:** `429 Too Many Requests` with `Retry-After` header (seconds).

### 5.5 Endpoints

All paths relative to `https://api.box.com/2.0` unless noted.

#### File Operations

**Upload a File (up to 50 MB)**
```
POST https://upload.box.com/api/2.0/files/content
```
- **Content-Type:** `multipart/form-data`
- **Form Fields:**
  - `attributes`: JSON string `{"name": "file.pdf", "parent": {"id": "0"}}` (id "0" = root folder)
  - `file`: Binary file data
- **Response:** `{ "entries": [File] }` (File resource with id, name, size, etc.)

**Upload New Version of a File**
```
POST https://upload.box.com/api/2.0/files/{file-id}/content
```
- Same multipart format. Replaces file content while preserving file ID.
- **Headers:** `If-Match: {etag}` (optional, for conflict detection)
- **Response:** `{ "entries": [File] }`

**Chunked Upload -- Create Session**
```
POST https://upload.box.com/api/2.0/files/upload_sessions
```
- **Request Body:**
  ```json
  {
    "file_size": 104857600,
    "file_name": "large-file.zip",
    "folder_id": "0"
  }
  ```
- **Response:** `UploadSession` with `id`, `session_endpoints` (upload_part, commit, abort, status, list_parts), `part_size`, `total_parts`, `num_parts_processed`.

**Chunked Upload -- Upload Part**
```
PUT https://upload.box.com/api/2.0/files/upload_sessions/{session-id}
```
- **Headers:**
  - `Content-Range: bytes {start}-{end}/{total}`
  - `Digest: sha={base64-SHA1-of-part}`
  - `Content-Type: application/octet-stream`
- **Request Body:** Binary chunk data.
- **Part Size:** Determined by the `part_size` field from the session creation response (typically 8 MB-64 MB).
- **Response:** `{ "part": Part }` with `part_id`, `offset`, `size`, `sha1`.

**Chunked Upload -- Commit**
```
POST https://upload.box.com/api/2.0/files/upload_sessions/{session-id}/commit
```
- **Headers:** `Digest: sha={base64-SHA1-of-entire-file}`
- **Request Body:**
  ```json
  {
    "parts": [
      { "part_id": "...", "offset": 0, "size": 8388608, "sha1": "..." },
      ...
    ]
  }
  ```
- **Response:** `{ "entries": [File] }` or `202 Accepted` if processing.

**Chunked Upload -- Abort**
```
DELETE https://upload.box.com/api/2.0/files/upload_sessions/{session-id}
```
- **Response:** `204 No Content`.

**Download a File**
```
GET /files/{file-id}/content
```
- **Response:** `302 Found` redirect to a temporary download URL.
- **Headers:** Supports `Range` for partial downloads.

**Get File Information**
```
GET /files/{file-id}
```
- **Query Parameters:** `fields=id,name,size,modified_at,shared_link,parent,path_collection`
- **Response:** `File` resource.

**Search Files**
```
GET /search
```
- **Query Parameters:**
  - `query` (required): Search string
  - `type`: `file` or `folder`
  - `file_extensions`: Comma-separated list (e.g., `pdf,docx`)
  - `ancestor_folder_ids`: Restrict to specific folders
  - `content_types`: `name`, `description`, `file_content`, `comments`, `tag`
  - `created_at_range`: Date range filter
  - `updated_at_range`: Date range filter
  - `size_range`: File size range in bytes
  - `limit`: Max 200
  - `offset`: For pagination
- **Response:** `{ "entries": [Item], "total_count": int, "limit": int, "offset": int }`

**Copy a File**
```
POST /files/{file-id}/copy
```
- **Request Body:**
  ```json
  {
    "parent": { "id": "destination-folder-id" },
    "name": "Copy of file.pdf"
  }
  ```
- **Response:** `File` resource of the copy.

**Move / Rename a File**
```
PUT /files/{file-id}
```
- **Request Body:**
  ```json
  {
    "parent": { "id": "new-folder-id" },
    "name": "renamed-file.pdf"
  }
  ```
- **Response:** Updated `File` resource.

**Delete a File**
```
DELETE /files/{file-id}
```
- **Response:** `204 No Content`. Moves to trash by default.

**Permanently Delete (Trash)**
```
DELETE /files/{file-id}/trash
```
- **Response:** `204 No Content`.

**Create Shared Link**
```
PUT /files/{file-id}
```
- **Request Body:**
  ```json
  {
    "shared_link": {
      "access": "open|company|collaborators",
      "password": "optional",
      "unshared_at": "2026-12-31T00:00:00Z",
      "permissions": {
        "can_download": true,
        "can_preview": true,
        "can_edit": false
      }
    }
  }
  ```
- **Response:** `File` resource with `shared_link` object containing `url`, `download_url`, `effective_access`.

#### Folder Operations

**Create a Folder**
```
POST /folders
```
- **Request Body:**
  ```json
  {
    "name": "New Folder",
    "parent": { "id": "0" }
  }
  ```
- **Response:** `Folder` resource.

**Get Folder Items**
```
GET /folders/{folder-id}/items
```
- **Query Parameters:** `fields`, `limit` (max 1000), `offset`, `sort` (id, name, date, size), `direction` (ASC, DESC)
- **Response:** `{ "entries": [Item], "total_count": int }`

**Get Folder Info**
```
GET /folders/{folder-id}
```
- **Response:** `Folder` resource.

**Update a Folder**
```
PUT /folders/{folder-id}
```
- **Request Body:** `{ "name": "Renamed Folder", "description": "..." }`
- **Response:** Updated `Folder` resource.

**Delete a Folder**
```
DELETE /folders/{folder-id}
```
- **Query Parameters:** `recursive=true` (required if folder has contents)
- **Response:** `204 No Content`.

**Share a Folder**
```
PUT /folders/{folder-id}
```
- Same shared link pattern as files.

### 5.6 Webhooks / Real-time Notifications

**Create Webhook**
```
POST /webhooks
```
- **Request Body:**
  ```json
  {
    "target": {
      "id": "folder-id",
      "type": "folder"
    },
    "address": "https://esmer.app/webhooks/box",
    "triggers": [
      "FILE.UPLOADED",
      "FILE.DOWNLOADED",
      "FILE.TRASHED",
      "FILE.DELETED",
      "FILE.RESTORED",
      "FILE.COPIED",
      "FILE.MOVED",
      "FILE.RENAMED",
      "FOLDER.CREATED",
      "FOLDER.RENAMED",
      "FOLDER.DELETED",
      "FOLDER.MOVED",
      "FOLDER.COPIED",
      "COMMENT.CREATED",
      "COLLABORATION.CREATED",
      "SHARED_LINK.CREATED",
      "SHARED_LINK.DELETED",
      "TASK_ASSIGNMENT.CREATED",
      "TASK_ASSIGNMENT.UPDATED",
      "METADATA_INSTANCE.CREATED",
      "METADATA_INSTANCE.UPDATED",
      "METADATA_INSTANCE.DELETED",
      "SIGN_REQUEST.COMPLETED",
      "SIGN_REQUEST.DECLINED"
    ]
  }
  ```
- **Response:** `Webhook` resource with `id`, `address`, `triggers`.
- **Verification:** Box includes `BOX-DELIVERY-ID`, `BOX-DELIVERY-TIMESTAMP`, and `BOX-SIGNATURE-PRIMARY` / `BOX-SIGNATURE-SECONDARY` headers. Verify using HMAC-SHA256 with your webhook signature key.
- **Maximum Webhooks:** 1000 per application.

**List Webhooks**
```
GET /webhooks
```

**Delete Webhook**
```
DELETE /webhooks/{webhook-id}
```

### 5.7 Error Handling

| HTTP Code | Error Code | Description | Retry? |
|---|---|---|---|
| 400 | `bad_request` | Invalid request | No |
| 401 | `unauthorized` | Token expired/invalid | Refresh token, retry |
| 403 | `access_denied_insufficient_permissions` | Insufficient permissions | No |
| 404 | `not_found` | Item not found | No |
| 405 | `method_not_allowed` | HTTP method not allowed | No |
| 409 | `conflict` | Name conflict | No (use different name) |
| 409 | `item_name_in_use` | Duplicate name in folder | No |
| 412 | `precondition_failed` | ETag mismatch | Retry with fresh ETag |
| 429 | `rate_limit_exceeded` | Rate limited | Yes, respect `Retry-After` |
| 500 | `internal_server_error` | Server error | Yes, exponential backoff |
| 503 | `unavailable` | Service temporarily down | Yes, exponential backoff |

### 5.8 Upload / Download Details

**Direct Upload:**
- Maximum 50 MB
- Multipart form-data with `attributes` JSON + `file` binary
- Endpoint: `https://upload.box.com/api/2.0/files/content`

**Chunked Upload (Resumable):**
1. Create session: specifies `file_size` and `file_name`
2. Upload parts: part size dictated by server (typically 8-64 MB). Include SHA-1 digest per part.
3. Commit: include SHA-1 of entire file. List all parts.
4. Abort: DELETE the session to cancel.
- Maximum file size: 50 GB (150 GB for Box admin-enabled accounts)
- Session expiry: 7 days
- Parts can be uploaded in parallel

**Download:**
- GET returns 302 redirect to temporary URL
- Temporary URL valid for minutes
- Supports `Range` header
- Use `representations` API for preview/thumbnail without full download

**Preflight Check:**
```
OPTIONS /files/content
```
- Request Body: `{ "name": "file.pdf", "parent": { "id": "0" }, "size": 1048576 }`
- Returns `200` if upload will succeed, or error with reason (quota, name conflict, etc.)

### 5.9 Mobile-Specific Notes

- **Thumbnails:** `GET /files/{id}/thumbnail.{extension}` where extension is `png` or `jpg`. Query parameters: `min_width`, `min_height`, `max_width`, `max_height` (between 32 and 2048 px). Returns `200` with image or `202` if thumbnail is being generated (retry after a short delay).
- **File Representations (Previews):** `GET /files/{id}?fields=representations` returns available representations (PDF, text, thumbnail, extracted text). Request a specific representation via the `info_url` and then `content.url_template`.
- **Shared Links:** Box shared links support password protection and expiry, useful for secure mobile sharing.
- **Preflight Check:** Always call preflight before upload on mobile to avoid wasting bandwidth on doomed uploads.
- **Collaboration:** Add collaborators via `POST /collaborations` with `{ "item": {"id": "...", "type": "folder"}, "accessible_by": {"type": "user", "login": "email"}, "role": "editor|viewer|..." }`.

---

## 6. Nextcloud (WebDAV / OCS API)

### 6.1 Service Overview

Nextcloud is a self-hosted, open-source file sync and collaboration platform. For Esmer, Nextcloud integration serves privacy-conscious users and organizations that run their own cloud infrastructure. Key capabilities include file storage on user-controlled servers, sharing with internal/external users, and user management. Unlike other services in this category, Nextcloud's base URL varies per deployment.

### 6.2 Authentication

**Method 1: Username + Password (Basic Auth or App Passwords)**
- HTTP Basic Authentication with username and password (or app-specific password)
- App passwords generated in Nextcloud Settings > Security > App passwords

**Method 2: OAuth 2.0**
- Available if the OAuth 2.0 app is enabled on the Nextcloud instance

| Parameter | Value |
|---|---|
| Authorization URL | `https://{nextcloud-host}/index.php/apps/oauth2/authorize` |
| Token URL | `https://{nextcloud-host}/index.php/apps/oauth2/api/v1/token` |
| Grant Type | `authorization_code` |

**Recommended for Esmer:** Use OAuth 2.0 when available, falling back to app passwords (never store actual user passwords). Store the Nextcloud server URL per user since it varies.

### 6.3 Base URL

WebDAV (file operations):
```
https://{nextcloud-host}/remote.php/dav/files/{username}
```

OCS (sharing, user management):
```
https://{nextcloud-host}/ocs/v2.php
```

Webdav (other):
```
https://{nextcloud-host}/remote.php/dav
```

### 6.4 Rate Limits

| Limit Type | Value |
|---|---|
| Default rate limiting | Configurable per instance (Nextcloud Brute Force Protection) |
| Typical defaults | No hard API rate limits (server capacity dependent) |
| File size limit | Configured per server (PHP `upload_max_filesize`, `post_max_size`) |
| Default PHP upload limit | 512 MB (configurable) |
| Chunked upload chunk size | Configurable, default 10 MB |

**Note:** Since Nextcloud is self-hosted, rate limits depend on the server configuration and capacity. Esmer should handle `503 Service Unavailable` gracefully.

### 6.5 Endpoints

#### File Operations (WebDAV)

All paths relative to `https://{host}/remote.php/dav/files/{username}`.

**Upload a File**
```
PUT /remote.php/dav/files/{username}/{path/to/file.pdf}
```
- **Headers:** `Content-Type: {mime_type}`
- **Request Body:** Raw binary file content.
- **Response:** `201 Created` or `204 No Content` (if overwriting).

**Chunked Upload (Nextcloud Chunking v2)**

1. **Create upload directory:**
```
MKCOL /remote.php/dav/uploads/{username}/{unique-upload-id}
```
- **Response:** `201 Created`.

2. **Upload chunks:**
```
PUT /remote.php/dav/uploads/{username}/{unique-upload-id}/{chunk-number}
```
- Chunk numbers are sequential (1, 2, 3, ...) or byte-offset based.
- **Request Body:** Binary chunk data.
- **Response:** `201 Created`.

3. **Assemble (move to final destination):**
```
MOVE /remote.php/dav/uploads/{username}/{unique-upload-id}/.file
```
- **Headers:** `Destination: https://{host}/remote.php/dav/files/{username}/{path/to/file.pdf}`
- **Response:** `201 Created` or `204 No Content`.

**Download a File**
```
GET /remote.php/dav/files/{username}/{path/to/file.pdf}
```
- **Response:** Binary file content with appropriate `Content-Type`.
- Supports `Range` header for partial downloads.

**Get File Properties (Metadata)**
```
PROPFIND /remote.php/dav/files/{username}/{path/to/file.pdf}
```
- **Headers:** `Depth: 0` (single file) or `Depth: 1` (directory listing)
- **Request Body (XML):**
  ```xml
  <?xml version="1.0"?>
  <d:propfind xmlns:d="DAV:" xmlns:oc="http://owncloud.org/ns" xmlns:nc="http://nextcloud.org/ns">
    <d:prop>
      <d:getlastmodified/>
      <d:getcontentlength/>
      <d:getcontenttype/>
      <d:getetag/>
      <oc:fileid/>
      <oc:permissions/>
      <oc:size/>
      <nc:has-preview/>
    </d:prop>
  </d:propfind>
  ```
- **Response:** `207 Multi-Status` XML with property values.

**List Folder Contents**
```
PROPFIND /remote.php/dav/files/{username}/{folder-path}/
```
- **Headers:** `Depth: 1`
- **Response:** `207 Multi-Status` XML listing all children.

**Copy a File**
```
COPY /remote.php/dav/files/{username}/{source-path}
```
- **Headers:**
  - `Destination: https://{host}/remote.php/dav/files/{username}/{dest-path}`
  - `Overwrite: T|F`
- **Response:** `201 Created` or `204 No Content`.

**Move a File**
```
MOVE /remote.php/dav/files/{username}/{source-path}
```
- **Headers:**
  - `Destination: https://{host}/remote.php/dav/files/{username}/{dest-path}`
  - `Overwrite: T|F`
- **Response:** `201 Created` or `204 No Content`.

**Delete a File**
```
DELETE /remote.php/dav/files/{username}/{path/to/file.pdf}
```
- **Response:** `204 No Content`. Moves to trash (if trash app enabled).

#### Folder Operations (WebDAV)

**Create a Folder**
```
MKCOL /remote.php/dav/files/{username}/{path/to/new-folder}
```
- **Response:** `201 Created`.

**Delete a Folder**
```
DELETE /remote.php/dav/files/{username}/{folder-path}/
```
- **Response:** `204 No Content`. Recursively deletes contents.

**Copy a Folder**
```
COPY /remote.php/dav/files/{username}/{source-folder}/
```
- **Headers:** `Destination: ...`, `Depth: infinity`
- **Response:** `201 Created`.

**Move a Folder**
```
MOVE /remote.php/dav/files/{username}/{source-folder}/
```
- Same pattern as file move.

#### Sharing (OCS API)

**Share a File/Folder**
```
POST /ocs/v2.php/apps/files_sharing/api/v1/shares
```
- **Headers:** `OCS-APIRequest: true`, `Content-Type: application/x-www-form-urlencoded`
- **Parameters:**
  - `path` (required): Path to file/folder, e.g., `/Documents/file.pdf`
  - `shareType` (required): `0` (user), `1` (group), `3` (public link), `4` (email), `6` (federated cloud share), `7` (circle), `10` (talk conversation)
  - `shareWith` (required for types 0, 1, 4, 6, 7, 10): Username, group name, email, etc.
  - `permissions` (optional): Bitmask: `1` (read), `2` (update), `4` (create), `8` (delete), `16` (share), `31` (all)
  - `password` (optional): Password for public links
  - `expireDate` (optional): Format `YYYY-MM-DD`
- **Response:** `200 OK` with XML/JSON containing share `id`, `url`, `token`.

**Get Shares for a File**
```
GET /ocs/v2.php/apps/files_sharing/api/v1/shares?path={path}
```
- **Headers:** `OCS-APIRequest: true`
- **Response:** List of share objects.

**Delete a Share**
```
DELETE /ocs/v2.php/apps/files_sharing/api/v1/shares/{share-id}
```

**Update a Share**
```
PUT /ocs/v2.php/apps/files_sharing/api/v1/shares/{share-id}
```
- **Parameters:** `permissions`, `password`, `expireDate`, `note`

#### User Management (OCS API)

**Create a User**
```
POST /ocs/v2.php/cloud/users
```
- **Parameters:** `userid`, `password`, `email`, `groups[]`, `displayName`, `quota`

**Get User Info**
```
GET /ocs/v2.php/cloud/users/{userid}
```

**Edit User**
```
PUT /ocs/v2.php/cloud/users/{userid}
```
- **Parameters:** `key` (email, quota, displayname, phone, address, website, twitter, password), `value`

**Delete User**
```
DELETE /ocs/v2.php/cloud/users/{userid}
```

**List Users**
```
GET /ocs/v2.php/cloud/users
```
- **Parameters:** `search`, `limit`, `offset`

### 6.6 Webhooks / Real-time Notifications

Nextcloud supports activity notifications and can be extended with the **Webhooks** app (community-maintained) or the **Flow/Workflow engine**.

**Activity API:**
```
GET /ocs/v2.php/apps/activity/api/v2/activity
```
- **Parameters:** `since` (activity ID), `limit`, `object_type`, `object_id`, `sort` (asc/desc)
- **Description:** Poll for recent file changes, shares, comments, etc.

**Nextcloud Flow (Automated Workflows):**
- Server-side workflows that trigger on file events (create, update, tag change)
- Can be configured to call external webhooks via the **Webhook** action in Flow

**Push Notifications (Nextcloud Notifications App):**
```
GET /ocs/v2.php/apps/notifications/api/v2/notifications
```
- For mobile: Nextcloud supports push notifications via the **Push Notifications** app using UnifiedPush or proprietary push.

### 6.7 Error Handling

| HTTP Code | Description | Retry? |
|---|---|---|
| 400 | Bad request | No |
| 401 | Authentication required / invalid credentials | Re-authenticate |
| 403 | Forbidden (insufficient permissions) | No |
| 404 | File/folder not found | No |
| 405 | Method not allowed (e.g., MKCOL on existing dir) | No |
| 409 | Conflict (parent directory does not exist) | No, create parents first |
| 412 | Precondition failed (ETag mismatch) | Retry with fresh ETag |
| 423 | Locked (file is locked for editing) | Yes, retry after delay |
| 507 | Insufficient storage | No |
| 503 | Service unavailable | Yes, exponential backoff |

**WebDAV Errors:** Returned as XML `<d:error>` elements with `<s:message>` and `<s:exception>` details.

### 6.8 Upload / Download Details

**Simple Upload (PUT):**
- Maximum depends on server PHP configuration (`upload_max_filesize`, `post_max_size`)
- Typical default: 512 MB
- Single PUT with raw binary body

**Chunked Upload (v2):**
1. MKCOL to create upload staging directory
2. PUT chunks sequentially or in parallel
3. MOVE `.file` pseudo-resource to final destination to assemble
- Chunk size: configurable (default 10 MB, recommended 5-50 MB)
- No inherent file size limit beyond server filesystem/quota
- Resumable: re-upload only missing chunks

**Download:**
- Simple GET request
- Supports `Range` header for partial downloads
- ETags provided for caching

### 6.9 Mobile-Specific Notes

- **Thumbnails:** `GET /index.php/apps/files/api/v1/thumbnail/{width}/{height}/{path}` or via WebDAV PROPFIND with `nc:has-preview` to check availability, then `GET /index.php/core/preview?fileId={id}&x={w}&y={h}`.
- **Server Discovery:** Esmer must store a per-user Nextcloud server URL. Use `GET /.well-known/webdav` for WebDAV endpoint discovery. Use `GET /status.php` to verify it is a Nextcloud instance and get version info.
- **Self-Signed Certificates:** Many self-hosted Nextcloud instances use self-signed TLS certificates. Esmer should warn users but allow certificate pinning.
- **Variable Server Capacity:** Upload chunk sizes should be adaptive based on upload speed and server responsiveness.
- **OCS Format:** Append `?format=json` to OCS API calls to receive JSON instead of XML (easier to parse on mobile).

---

## 7. AWS S3 (REST API)

### 7.1 Service Overview

Amazon S3 (Simple Storage Service) is an object storage service designed for high durability, availability, and scalability. For Esmer, S3 serves as infrastructure-grade storage for users or organizations that use AWS. Key use cases include storing large artifacts (backups, media, datasets), serving as a backend for file attachments that need high availability, and integrating with other AWS services (Lambda triggers, CloudFront CDN). S3 uses a flat object model (bucket + key) rather than a traditional file/folder hierarchy.

### 7.2 Authentication

**Method:** AWS Signature Version 4 (SigV4)

S3 does not use OAuth. Instead, it uses AWS IAM credentials:

| Credential Type | Description |
|---|---|
| Access Key ID + Secret Access Key | Long-term IAM user credentials |
| Temporary Security Credentials | From AWS STS (AssumeRole), includes session token |
| IAM Roles (for EC2/Lambda) | Instance profiles, no explicit keys needed |

**Signature Process (AWS SigV4):**
1. Create a canonical request (method, path, query string, headers, payload hash)
2. Create string-to-sign (algorithm, date, credential scope, canonical request hash)
3. Calculate signing key: HMAC-SHA256 chain of date, region, service, "aws4_request" with secret key
4. Add `Authorization` header:
   ```
   AWS4-HMAC-SHA256 Credential={access-key}/{date}/{region}/s3/aws4_request, SignedHeaders={headers}, Signature={signature}
   ```

**Required IAM Permissions (Policy Actions):**

| Action | Purpose |
|---|---|
| `s3:PutObject` | Upload files |
| `s3:GetObject` | Download files |
| `s3:DeleteObject` | Delete files |
| `s3:ListBucket` | List objects in a bucket |
| `s3:ListAllMyBuckets` | List all buckets |
| `s3:CreateBucket` | Create new buckets |
| `s3:DeleteBucket` | Delete buckets |
| `s3:GetBucketLocation` | Get bucket region |
| `s3:PutObjectAcl` | Set object permissions |
| `s3:GetObjectAcl` | Read object permissions |

**Recommended IAM Policy for Esmer:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket",
        "s3:ListAllMyBuckets",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::esmer-*",
        "arn:aws:s3:::esmer-*/*"
      ]
    }
  ]
}
```

**Pre-Signed URLs:** For mobile clients that should not hold AWS credentials directly, generate pre-signed URLs server-side (valid for up to 7 days for SigV4).

### 7.3 Base URL

**Path-style (legacy, deprecated for new buckets):**
```
https://s3.{region}.amazonaws.com/{bucket-name}
```

**Virtual-hosted style (recommended):**
```
https://{bucket-name}.s3.{region}.amazonaws.com
```

**Global endpoint (for ListBuckets only):**
```
https://s3.amazonaws.com
```

Common regions: `us-east-1`, `us-west-2`, `eu-west-1`, `ap-southeast-1`, etc.

### 7.4 Rate Limits

| Limit Type | Value |
|---|---|
| PUT/COPY/POST/DELETE requests per prefix | 3,500 per second |
| GET/HEAD requests per prefix | 5,500 per second |
| Object size (single PUT) | 5 GB |
| Object size (multipart upload) | 5 TB |
| Minimum part size (multipart) | 5 MB (except last part) |
| Maximum parts per upload | 10,000 |
| Maximum buckets per account | 100 (soft limit, can request increase) |
| Object key length | 1,024 bytes (UTF-8) |

**Throttling:** S3 returns `503 Slow Down` when request rate is too high. Use exponential backoff. S3 automatically scales to support higher request rates; temporary throttling during scaling is normal.

### 7.5 Endpoints

#### Bucket Operations

**List All Buckets**
```
GET /
Host: s3.amazonaws.com
```
- **Response:** XML `<ListAllMyBucketsResult>` with `<Buckets><Bucket><Name>` and `<CreationDate>`.

**Create a Bucket**
```
PUT /
Host: {bucket-name}.s3.{region}.amazonaws.com
```
- **Request Body (for non-us-east-1):**
  ```xml
  <CreateBucketConfiguration>
    <LocationConstraint>{region}</LocationConstraint>
  </CreateBucketConfiguration>
  ```
- **Response:** `200 OK` with `Location` header.

**Delete a Bucket**
```
DELETE /
Host: {bucket-name}.s3.{region}.amazonaws.com
```
- **Prerequisite:** Bucket must be empty.
- **Response:** `204 No Content`.

**Get Bucket Location**
```
GET /?location
Host: {bucket-name}.s3.amazonaws.com
```
- **Response:** XML with `<LocationConstraint>` element.

#### Object (File) Operations

**Upload an Object (PUT, up to 5 GB)**
```
PUT /{key}
Host: {bucket-name}.s3.{region}.amazonaws.com
```
- **Headers:**
  - `Content-Type: {mime_type}`
  - `Content-Length: {size}`
  - `Content-MD5: {base64-MD5}` (optional, for integrity)
  - `x-amz-storage-class: STANDARD|INTELLIGENT_TIERING|GLACIER|...` (optional)
  - `x-amz-server-side-encryption: AES256|aws:kms` (optional)
  - `x-amz-meta-{key}: {value}` (custom metadata)
- **Request Body:** Raw binary.
- **Response:** `200 OK` with `ETag` header (MD5 of object).

**Multipart Upload -- Initiate**
```
POST /{key}?uploads
Host: {bucket-name}.s3.{region}.amazonaws.com
```
- **Headers:** Same as PUT (Content-Type, storage class, encryption, metadata).
- **Response:** XML `<InitiateMultipartUploadResult>` with `<UploadId>`.

**Multipart Upload -- Upload Part**
```
PUT /{key}?partNumber={part-number}&uploadId={upload-id}
Host: {bucket-name}.s3.{region}.amazonaws.com
```
- **Request Body:** Part binary data (min 5 MB, max 5 GB per part; except last part).
- **Response:** `200 OK` with `ETag` header for the part.

**Multipart Upload -- Complete**
```
POST /{key}?uploadId={upload-id}
Host: {bucket-name}.s3.{region}.amazonaws.com
```
- **Request Body:**
  ```xml
  <CompleteMultipartUpload>
    <Part>
      <PartNumber>1</PartNumber>
      <ETag>"etag1"</ETag>
    </Part>
    <Part>
      <PartNumber>2</PartNumber>
      <ETag>"etag2"</ETag>
    </Part>
  </CompleteMultipartUpload>
  ```
- **Response:** XML `<CompleteMultipartUploadResult>` with `<Location>`, `<Bucket>`, `<Key>`, `<ETag>`.

**Multipart Upload -- Abort**
```
DELETE /{key}?uploadId={upload-id}
```
- **Response:** `204 No Content`. Cleans up uploaded parts.

**Multipart Upload -- List Parts**
```
GET /{key}?uploadId={upload-id}
```
- **Response:** XML listing uploaded parts with ETags and sizes.

**Download an Object**
```
GET /{key}
Host: {bucket-name}.s3.{region}.amazonaws.com
```
- **Headers:**
  - `Range: bytes={start}-{end}` (optional, partial download)
  - `If-None-Match: {etag}` (conditional, for caching)
  - `If-Modified-Since: {date}` (conditional)
- **Response:** `200 OK` (full) or `206 Partial Content` (range) with binary body.

**Head Object (Get Metadata Only)**
```
HEAD /{key}
Host: {bucket-name}.s3.{region}.amazonaws.com
```
- **Response:** Headers only (`Content-Type`, `Content-Length`, `Last-Modified`, `ETag`, custom `x-amz-meta-*`).

**Copy an Object**
```
PUT /{destination-key}
Host: {dest-bucket}.s3.{region}.amazonaws.com
```
- **Headers:** `x-amz-copy-source: /{source-bucket}/{source-key}`
- **Response:** XML `<CopyObjectResult>` with `<ETag>` and `<LastModified>`.
- **Note:** For objects > 5 GB, must use multipart copy (`UploadPartCopy`).

**Delete an Object**
```
DELETE /{key}
Host: {bucket-name}.s3.{region}.amazonaws.com
```
- **Response:** `204 No Content`.

**Delete Multiple Objects**
```
POST /?delete
Host: {bucket-name}.s3.{region}.amazonaws.com
```
- **Request Body:**
  ```xml
  <Delete>
    <Object><Key>file1.pdf</Key></Object>
    <Object><Key>file2.pdf</Key></Object>
  </Delete>
  ```
- **Response:** XML `<DeleteResult>` with `<Deleted>` and `<Error>` entries.

**List Objects (v2)**
```
GET /?list-type=2
Host: {bucket-name}.s3.{region}.amazonaws.com
```
- **Query Parameters:**
  - `prefix` (filter by key prefix, simulates folder browsing)
  - `delimiter` (typically `/`, for folder-like listing)
  - `max-keys` (default 1000, max 1000)
  - `continuation-token` (for pagination)
  - `start-after` (start listing after this key)
- **Response:** XML `<ListBucketResult>` with `<Contents>` (objects) and `<CommonPrefixes>` (virtual folders).

**Search (List with Prefix):**
S3 does not have a full-text search API. Searching is done by prefix filtering with `?list-type=2&prefix={search-term}`. For content search, integrate with Amazon Kendra or build a custom search index.

#### Pre-Signed URLs

**Generate Pre-Signed URL (server-side, using AWS SDK):**
- For uploads: Pre-sign a PUT request
- For downloads: Pre-sign a GET request
- Maximum validity: 7 days (604800 seconds) with SigV4
- Include in URL: `X-Amz-Algorithm`, `X-Amz-Credential`, `X-Amz-Date`, `X-Amz-Expires`, `X-Amz-SignedHeaders`, `X-Amz-Signature`

#### Folder Operations (Virtual)

S3 has no native folder concept. Folders are simulated via key prefixes and zero-byte objects:

**Create a Folder (virtual)**
```
PUT /{folder-name}/
Host: {bucket-name}.s3.{region}.amazonaws.com
```
- **Content-Length:** `0`
- Creates a zero-byte object with a trailing `/`.

**Delete a Folder:** List all objects with the prefix, then delete them all.

### 7.6 Webhooks / Real-time Notifications

**S3 Event Notifications:**

S3 can send notifications to SNS, SQS, or Lambda on object events.

**Supported Events:**
- `s3:ObjectCreated:*` (Put, Post, Copy, CompleteMultipartUpload)
- `s3:ObjectRemoved:*` (Delete, DeleteMarkerCreated)
- `s3:ObjectRestore:*` (Post, Completed, Delete)
- `s3:ReducedRedundancyLostObject`
- `s3:Replication:*`
- `s3:LifecycleExpiration:*`
- `s3:LifecycleTransition`
- `s3:IntelligentTiering`
- `s3:ObjectTagging:*`
- `s3:ObjectAcl:Put`

**Configuration (via PutBucketNotificationConfiguration):**
```
PUT /?notification
Host: {bucket-name}.s3.{region}.amazonaws.com
```
- **Request Body:**
  ```xml
  <NotificationConfiguration>
    <TopicConfiguration>
      <Id>esmer-upload-notify</Id>
      <Topic>arn:aws:sns:{region}:{account-id}:esmer-notifications</Topic>
      <Event>s3:ObjectCreated:*</Event>
      <Filter>
        <S3Key>
          <FilterRule>
            <Name>prefix</Name>
            <Value>uploads/</Value>
          </FilterRule>
          <FilterRule>
            <Name>suffix</Name>
            <Value>.pdf</Value>
          </FilterRule>
        </S3Key>
      </Filter>
    </TopicConfiguration>
  </NotificationConfiguration>
  ```

**Amazon EventBridge Integration:** Alternatively, enable EventBridge notifications for the bucket to receive all events in EventBridge with custom routing rules.

### 7.7 Error Handling

| HTTP Code | Error Code | Description | Retry? |
|---|---|---|---|
| 400 | `InvalidArgument` | Invalid request parameter | No |
| 400 | `InvalidBucketName` | Invalid bucket name | No |
| 400 | `EntityTooLarge` | Object exceeds 5 GB (single PUT) | No, use multipart |
| 403 | `AccessDenied` | IAM permission denied | No |
| 403 | `SignatureDoesNotMatch` | Invalid signature | No, fix credentials |
| 404 | `NoSuchBucket` | Bucket does not exist | No |
| 404 | `NoSuchKey` | Object does not exist | No |
| 409 | `BucketAlreadyExists` | Bucket name taken globally | No |
| 409 | `BucketNotEmpty` | Cannot delete non-empty bucket | No, empty first |
| 412 | `PreconditionFailed` | Conditional request failed | No |
| 416 | `InvalidRange` | Invalid Range header | No |
| 429 | N/A | Throttled (rare) | Yes, exponential backoff |
| 500 | `InternalError` | AWS server error | Yes, exponential backoff |
| 503 | `SlowDown` | Request rate too high | Yes, exponential backoff |
| 503 | `ServiceUnavailable` | AWS temporarily unavailable | Yes, exponential backoff |

**Error Response Format:**
```xml
<Error>
  <Code>NoSuchKey</Code>
  <Message>The specified key does not exist.</Message>
  <Key>my-file.pdf</Key>
  <RequestId>4442587FB7D0A2F9</RequestId>
</Error>
```

### 7.8 Upload / Download Details

**Single PUT Upload:**
- Maximum 5 GB per request
- Recommended for files under 100 MB
- Include `Content-MD5` for integrity verification

**Multipart Upload:**
- Required for files > 5 GB, recommended for files > 100 MB
- Part size: minimum 5 MB (except last part), maximum 5 GB
- Maximum 10,000 parts, maximum total 5 TB
- Parts can be uploaded in parallel
- Incomplete multipart uploads consume storage; use lifecycle rules to auto-abort stale uploads
- `ListMultipartUploads` to find in-progress uploads

**Pre-Signed Upload (for mobile):**
1. Mobile client requests pre-signed URL from Esmer backend
2. Backend generates pre-signed PUT URL with AWS SDK
3. Mobile client uploads directly to S3 using the pre-signed URL (no AWS credentials on device)
4. Backend is notified via S3 event notification

**Download:**
- GET with optional `Range` header for partial/resumable downloads
- Support for conditional GETs (`If-None-Match`, `If-Modified-Since`)
- Use CloudFront CDN for faster global downloads
- Pre-signed GET URLs for secure temporary access

**Transfer Acceleration:**
- Enable via `PUT /?accelerate` with `<AccelerateConfiguration><Status>Enabled</Status></AccelerateConfiguration>`
- Use endpoint: `{bucket}.s3-accelerate.amazonaws.com`
- Routes uploads through AWS CloudFront edge locations for faster transfers

### 7.9 Mobile-Specific Notes

- **No Native Thumbnails:** S3 does not generate thumbnails. Implement via Lambda@Edge or a CloudFront function that resizes images on-the-fly, or pre-generate thumbnails on upload.
- **Pre-Signed URLs:** Essential for mobile. Never embed AWS credentials in mobile apps. Generate short-lived pre-signed URLs (15-60 minutes) for both upload and download.
- **Transfer Acceleration:** Enable for mobile users uploading from geographically distant locations.
- **Multipart for Reliability:** Use multipart upload for any file > 10 MB on mobile to handle unreliable connections. Each part can be independently retried.
- **Content-MD5:** Include `Content-MD5` header on mobile uploads to detect corruption from unreliable networks.
- **S3 Select:** For CSV/JSON files, use `POST /{key}?select&select-type=2` to query file contents server-side and return only matching rows, reducing bandwidth.
- **Storage Classes:** Use `INTELLIGENT_TIERING` for user-uploaded files with unpredictable access patterns to optimize costs.

---

## 8. Google Cloud Storage (JSON API v1)

### 8.1 Service Overview

Google Cloud Storage (GCS) is Google's object storage service, analogous to AWS S3 but within the Google Cloud ecosystem. For Esmer, GCS provides infrastructure-grade storage for organizations using Google Cloud. Key use cases include storing large datasets, media assets, AI model outputs, and serving as a backend for applications. GCS offers strong consistency, multiple storage classes, and tight integration with other Google Cloud services.

### 8.2 Authentication

**Method 1: OAuth 2.0 (for user-facing operations)**

| Parameter | Value |
|---|---|
| Authorization URL | `https://accounts.google.com/o/oauth2/v2/auth` |
| Token URL | `https://oauth2.googleapis.com/token` |
| Grant Type | `authorization_code` |

**Required Scopes:**

| Scope | Purpose |
|---|---|
| `https://www.googleapis.com/auth/devstorage.full_control` | Full control of GCS resources |
| `https://www.googleapis.com/auth/devstorage.read_write` | Read/write access to GCS resources |
| `https://www.googleapis.com/auth/devstorage.read_only` | Read-only access |
| `https://www.googleapis.com/auth/cloud-platform` | Full access to all Google Cloud resources |

**Method 2: Service Account (for server-to-server)**
- Create a service account in Google Cloud Console
- Download JSON key file containing `client_email`, `private_key`, `token_uri`
- Generate a signed JWT and exchange for access token at `https://oauth2.googleapis.com/token`
- Grant the service account appropriate IAM roles on the bucket

**IAM Roles:**

| Role | Purpose |
|---|---|
| `roles/storage.objectViewer` | Read objects |
| `roles/storage.objectCreator` | Create objects |
| `roles/storage.objectAdmin` | Full control over objects |
| `roles/storage.admin` | Full control over buckets and objects |

**Method 3: Signed URLs (for mobile/temporary access)**
- Generate V4 signed URLs (maximum validity: 7 days)
- Signed with service account private key
- Include in URL: `X-Goog-Algorithm`, `X-Goog-Credential`, `X-Goog-Date`, `X-Goog-Expires`, `X-Goog-SignedHeaders`, `X-Goog-Signature`

### 8.3 Base URL

JSON API:
```
https://storage.googleapis.com/storage/v1
```

Upload endpoint:
```
https://storage.googleapis.com/upload/storage/v1
```

Download endpoint (direct):
```
https://storage.googleapis.com/{bucket-name}/{object-name}
```

XML API (S3-compatible):
```
https://storage.googleapis.com
```

### 8.4 Rate Limits

| Limit Type | Value |
|---|---|
| Read requests per second (per bucket) | 50,000 |
| Write requests per second (per bucket) | 5,000 |
| Object creation per second (per bucket) | 5,000 |
| Write requests per second (per object) | 1 |
| Object size (single upload) | 5 TB |
| Object size (simple upload, JSON API) | 5 MB |
| Object size (multipart upload, JSON API) | 5 MB |
| Object size (resumable upload) | 5 TB |
| Maximum object key length | 1,024 bytes (UTF-8) |
| Maximum buckets per project | 100 (soft limit) |
| Bucket creation/deletion rate | ~1 every 2 seconds |

**Throttling:** `429 Too Many Requests` or `503 Service Unavailable`. Use exponential backoff with jitter.

### 8.5 Endpoints

All paths relative to `https://storage.googleapis.com/storage/v1`.

#### Bucket Operations

**List Buckets**
```
GET /b?project={project-id}
```
- **Query Parameters:** `prefix`, `maxResults`, `pageToken`, `projection` (noAcl or full)
- **Response:** `{ "kind": "storage#buckets", "items": [Bucket], "nextPageToken": "..." }`

**Create a Bucket**
```
POST /b?project={project-id}
```
- **Request Body:**
  ```json
  {
    "name": "esmer-user-files",
    "location": "US",
    "storageClass": "STANDARD",
    "iamConfiguration": {
      "uniformBucketLevelAccess": { "enabled": true }
    }
  }
  ```
- **Response:** `Bucket` resource.

**Get a Bucket**
```
GET /b/{bucket-name}
```
- **Query Parameters:** `projection` (noAcl or full)
- **Response:** `Bucket` resource.

**Update a Bucket**
```
PATCH /b/{bucket-name}
```
- **Request Body:** Partial `Bucket` resource with fields to update.
- **Response:** Updated `Bucket` resource.

**Delete a Bucket**
```
DELETE /b/{bucket-name}
```
- **Prerequisite:** Bucket must be empty.
- **Response:** `204 No Content`.

#### Object Operations

**Upload an Object (Simple, up to 5 MB)**
```
POST /upload/storage/v1/b/{bucket-name}/o?uploadType=media&name={object-name}
```
- **Headers:** `Content-Type: {mime_type}`
- **Request Body:** Raw binary.
- **Response:** `Object` resource.

**Upload an Object (Multipart, metadata + data, up to 5 MB)**
```
POST /upload/storage/v1/b/{bucket-name}/o?uploadType=multipart
```
- **Content-Type:** `multipart/related; boundary={boundary}`
- **Body:**
  ```
  --{boundary}
  Content-Type: application/json; charset=UTF-8

  {"name": "file.pdf", "metadata": {"esmerTaskId": "abc123"}}
  --{boundary}
  Content-Type: application/pdf

  {binary data}
  --{boundary}--
  ```
- **Response:** `Object` resource.

**Upload an Object (Resumable, up to 5 TB)**

1. **Initiate:**
```
POST /upload/storage/v1/b/{bucket-name}/o?uploadType=resumable
```
- **Headers:**
  - `Content-Type: application/json; charset=UTF-8`
  - `X-Upload-Content-Type: {mime_type}`
  - `X-Upload-Content-Length: {total_bytes}` (optional but recommended)
- **Request Body:** Object metadata JSON.
- **Response:** `200 OK` with `Location` header containing the resumable upload URI.

2. **Upload data:**
```
PUT {resumable-upload-uri}
```
- **Headers:** `Content-Range: bytes {start}-{end}/{total}`
- **Chunk Size:** Minimum 256 KiB (262,144 bytes), must be a multiple of 256 KiB (except last chunk).
- **Response:** `308 Resume Incomplete` with `Range` header, or `200 OK` when complete.

3. **Check status:**
```
PUT {resumable-upload-uri}
```
- **Headers:** `Content-Range: bytes */{total}`
- **Response:** `308 Resume Incomplete` with `Range` indicating received bytes.

4. **Upload URI validity:** 1 week.

**Download an Object**
```
GET /b/{bucket-name}/o/{object-name}?alt=media
```
- **Headers:** `Range: bytes={start}-{end}` (optional)
- **Response:** Binary content.

**Alternative download (authenticated URL):**
```
GET https://storage.googleapis.com/{bucket-name}/{object-name}
```

**Get Object Metadata**
```
GET /b/{bucket-name}/o/{object-name}
```
- **Query Parameters:** `projection` (noAcl or full)
- **Response:** `Object` resource (name, bucket, generation, metageneration, contentType, size, md5Hash, crc32c, metadata, timeCreated, updated, etc.)

**List Objects**
```
GET /b/{bucket-name}/o
```
- **Query Parameters:**
  - `prefix` (filter by prefix)
  - `delimiter` (typically `/`)
  - `maxResults` (default 1000)
  - `pageToken`
  - `projection`
  - `versions` (include archived versions)
  - `startOffset`, `endOffset` (lexicographic range)
- **Response:** `{ "kind": "storage#objects", "items": [Object], "prefixes": [string], "nextPageToken": "..." }`

**Copy an Object**
```
POST /b/{source-bucket}/o/{source-object}/copyTo/b/{dest-bucket}/o/{dest-object}
```
- **Request Body:** Optional `Object` metadata for the destination.
- **Response:** `Object` resource of the copy.

**Compose Objects (Concatenate)**
```
POST /b/{bucket-name}/o/{dest-object}/compose
```
- **Request Body:**
  ```json
  {
    "sourceObjects": [
      { "name": "part1" },
      { "name": "part2" }
    ],
    "destination": { "contentType": "application/pdf" }
  }
  ```
- **Maximum:** 32 source objects per compose. Can chain for more.
- **Response:** `Object` resource.

**Update Object Metadata**
```
PATCH /b/{bucket-name}/o/{object-name}
```
- **Request Body:** Partial `Object` resource.
- **Response:** Updated `Object` resource.

**Delete an Object**
```
DELETE /b/{bucket-name}/o/{object-name}
```
- **Response:** `204 No Content`.

#### Access Control

**Set Object ACL**
```
POST /b/{bucket-name}/o/{object-name}/acl
```
- **Request Body:** `{ "entity": "user-email@example.com", "role": "READER|WRITER|OWNER" }`

**Make Object Public**
```
POST /b/{bucket-name}/o/{object-name}/acl
```
- **Request Body:** `{ "entity": "allUsers", "role": "READER" }`
- Public URL: `https://storage.googleapis.com/{bucket-name}/{object-name}`

### 8.6 Webhooks / Real-time Notifications

**Pub/Sub Notifications:**

GCS uses Cloud Pub/Sub for event notifications (not direct webhooks).

**Create Notification Configuration:**
```
POST /b/{bucket-name}/notificationConfigs
```
- **Request Body:**
  ```json
  {
    "topic": "projects/{project-id}/topics/{topic-name}",
    "event_types": [
      "OBJECT_FINALIZE",
      "OBJECT_METADATA_UPDATE",
      "OBJECT_DELETE",
      "OBJECT_ARCHIVE"
    ],
    "payload_format": "JSON_API_V1",
    "object_name_prefix": "uploads/"
  }
  ```
- **Response:** `Notification` resource with `id`.

**Event Types:**
- `OBJECT_FINALIZE` -- New object created or existing object overwritten
- `OBJECT_METADATA_UPDATE` -- Object metadata changed
- `OBJECT_DELETE` -- Object permanently deleted (or overwritten, if versioning off)
- `OBJECT_ARCHIVE` -- Live version becomes noncurrent (versioning enabled)

**List Notifications:**
```
GET /b/{bucket-name}/notificationConfigs
```

**Delete Notification:**
```
DELETE /b/{bucket-name}/notificationConfigs/{notification-id}
```

**Pub/Sub to Webhook Bridge:** To receive HTTP webhooks, create a Pub/Sub push subscription that forwards messages to your HTTPS endpoint:
```
gcloud pubsub subscriptions create esmer-gcs-sub \
  --topic={topic-name} \
  --push-endpoint=https://esmer.app/webhooks/gcs \
  --ack-deadline=60
```

### 8.7 Error Handling

| HTTP Code | Error Reason | Description | Retry? |
|---|---|---|---|
| 400 | `invalid` | Invalid request | No |
| 401 | `authError` | Invalid or expired token | Refresh token, retry |
| 403 | `forbidden` | Insufficient permissions | No |
| 403 | `rateLimitExceeded` | Rate limit exceeded | Yes, exponential backoff |
| 404 | `notFound` | Bucket or object not found | No |
| 409 | `conflict` | Bucket already exists or version conflict | No |
| 412 | `conditionNotMet` | Precondition (If-Match/If-Generation-Match) failed | Retry with updated metadata |
| 416 | `requestedRangeNotSatisfiable` | Invalid Range | No |
| 429 | `tooManyRequests` | Rate limited | Yes, exponential backoff |
| 500 | `internalError` | Google server error | Yes, exponential backoff |
| 503 | `serviceUnavailable` | Temporarily unavailable | Yes, exponential backoff |

**Error Response Format:**
```json
{
  "error": {
    "code": 404,
    "message": "No such object: bucket/object",
    "errors": [
      {
        "message": "No such object: bucket/object",
        "domain": "global",
        "reason": "notFound"
      }
    ]
  }
}
```

### 8.8 Upload / Download Details

**Simple Upload:**
- Maximum 5 MB
- Single POST with raw binary body
- Object name in query parameter

**Multipart Upload (JSON API):**
- Maximum 5 MB
- Two-part body: JSON metadata + binary content
- Object name in metadata JSON

**Resumable Upload:**
- For any file size up to 5 TB
- Chunk size: multiple of 256 KiB (recommended 8 MB+)
- URI valid for 1 week
- Can query upload status and resume from last byte received
- Supports unknown file size (use `Content-Range: bytes {start}-{end}/*` during upload, then `Content-Range: bytes {start}-{end}/{total}` for final chunk)

**Compose (Server-side Concatenation):**
- Combine up to 32 objects into one
- Useful for assembling chunks uploaded in parallel without using resumable upload
- No data transfer needed; performed server-side

**Download:**
- `alt=media` for binary download via JSON API
- Direct URL: `https://storage.googleapis.com/{bucket}/{object}`
- Supports `Range` header for partial downloads
- Supports conditional requests with `If-None-Match` (ETag) and `If-Modified-Since`

**Integrity:**
- GCS returns `x-goog-hash` header with `crc32c` and `md5` values
- Validate on client side after download

### 8.9 Mobile-Specific Notes

- **No Native Thumbnails:** Like S3, GCS does not generate thumbnails. Use Cloud Functions triggered by `OBJECT_FINALIZE` to create thumbnails, or use the Image CDN pattern with Cloud CDN + image transformation.
- **Signed URLs:** Generate V4 signed URLs (max 7 days) for mobile clients. Never embed service account keys in mobile apps. URL format: `https://storage.googleapis.com/{bucket}/{object}?X-Goog-Algorithm=GOOG4-RSA-SHA256&...`
- **Resumable Upload for Mobile:** Always use resumable upload for files > 5 MB on mobile. The 1-week URI validity accommodates users who start an upload on one session and complete it later.
- **Compose for Parallel Upload:** Upload multiple parts in parallel with unique keys, then compose them server-side. This can be faster than sequential resumable upload on mobile with limited upload bandwidth.
- **Storage Classes:** Use `STANDARD` for frequently accessed files and `NEARLINE` (30-day minimum) or `COLDLINE` (90-day minimum) for archival artifacts.
- **Integrity Verification:** Always verify the `crc32c` hash after upload/download on mobile to detect network corruption.
- **Public Access:** For sharing, either make the object public via ACL (permanent) or generate a signed URL with expiration (temporary, preferred for security).

---

## Cross-Service Comparison Matrix

### Authentication Methods

| Service | OAuth 2.0 | API Key | IAM/Service Account | Basic Auth |
|---|---|---|---|---|
| Google Drive | Yes | No | Yes (SA + JWT) | No |
| OneDrive | Yes | No | Yes (App-only) | No |
| SharePoint | Yes | No | Yes (App-only) | No |
| Dropbox | Yes | No | No | No |
| Box | Yes | No | Yes (JWT, CCG) | No |
| Nextcloud | Optional | No | No | Yes (App Password) |
| AWS S3 | No | No | Yes (SigV4) | No |
| Google Cloud Storage | Yes | No | Yes (SA + JWT) | No |

### Upload Size Limits

| Service | Simple Upload | Resumable/Chunked | Maximum File Size |
|---|---|---|---|
| Google Drive | 5 MB | 5 TB | 5 TB |
| OneDrive | 4 MB | 250 GB | 250 GB |
| SharePoint | 4 MB | 250 GB | 250 GB |
| Dropbox | 150 MB | 350 GB | 350 GB |
| Box | 50 MB | 50 GB | 50 GB (150 GB admin) |
| Nextcloud | Server-dependent | Server-dependent | Server-dependent |
| AWS S3 | 5 GB | 5 TB | 5 TB |
| Google Cloud Storage | 5 MB | 5 TB | 5 TB |

### Rate Limits Summary

| Service | Typical Limit | Throttle Response |
|---|---|---|
| Google Drive | 12,000 req / 100 sec / user | 403 or 429 |
| OneDrive | ~10,000 req / 10 min / app / tenant | 429 + Retry-After |
| SharePoint | ~10,000 req / 10 min / app / tenant | 429 + Retry-After |
| Dropbox | Dynamic (per endpoint) | 429 + Retry-After |
| Box | 10 req/sec/user, 1000 req/sec/app | 429 + Retry-After |
| Nextcloud | Server-dependent | 503 |
| AWS S3 | 3,500 PUT + 5,500 GET / sec / prefix | 503 SlowDown |
| Google Cloud Storage | 5,000 write + 50,000 read / sec / bucket | 429 or 503 |

### Webhook / Real-time Capabilities

| Service | Direct Webhook | Polling Alternative | Max Subscription Duration |
|---|---|---|---|
| Google Drive | Yes (watch API) | Changes API with startPageToken | 24 hours |
| OneDrive | Yes (Graph subscriptions) | Delta queries | 30 days |
| SharePoint | Yes (Graph subscriptions) | Delta queries | 30 days |
| Dropbox | Yes (App Console config) | list_folder/continue with cursor | Indefinite (app-level) |
| Box | Yes (API-managed) | Events API polling | Indefinite |
| Nextcloud | Via Flow/plugins | Activity API polling | Instance-dependent |
| AWS S3 | Via SNS/SQS/Lambda | ListObjectsV2 polling | Indefinite (bucket config) |
| Google Cloud Storage | Via Pub/Sub | ListObjects polling | Indefinite (bucket config) |

### Esmer Integration Priority

| Service | Priority | Rationale |
|---|---|---|
| Google Drive | P0 (Critical) | Largest consumer cloud storage user base |
| OneDrive | P0 (Critical) | Essential for Microsoft 365 enterprise users |
| Dropbox | P1 (High) | Large consumer and small business user base |
| SharePoint | P1 (High) | Enterprise document management backbone |
| Box | P2 (Medium) | Enterprise compliance-focused organizations |
| AWS S3 | P2 (Medium) | Developer/infrastructure use cases |
| Google Cloud Storage | P3 (Low) | Google Cloud-native organizations |
| Nextcloud | P3 (Low) | Self-hosted, privacy-conscious users |

---

## Appendix A: Recommended Esmer Implementation Patterns

### Unified File Interface

Esmer should expose a unified file interface across all providers:

```
EsmerFile {
  id: string              // Provider-specific file ID
  provider: string        // "gdrive" | "onedrive" | "sharepoint" | "dropbox" | "box" | "nextcloud" | "s3" | "gcs"
  name: string
  mimeType: string
  size: number            // bytes
  modifiedAt: ISO8601
  thumbnailUrl: string    // Provider-generated or Esmer-generated
  webViewUrl: string      // In-browser preview URL
  shareUrl: string        // Public share link (if shared)
  path: string            // Full path (provider-specific format)
  parentId: string        // Parent folder/container ID
}
```

### Upload Strategy Decision Tree

```
if fileSize <= 4MB:
    use simple upload (all providers support this)
elif fileSize <= 50MB:
    use simple upload (Dropbox, S3) or session upload (OneDrive, Box)
elif fileSize <= 150MB:
    use simple upload (Dropbox) or resumable upload (others)
else:
    always use resumable/chunked upload
    chunk size = max(provider_minimum, min(provider_maximum, adaptive_based_on_bandwidth))
```

### Mobile Upload Queue

For unreliable mobile connections:
1. Queue upload requests locally (SQLite/Realm)
2. Hash file content for deduplication
3. On connectivity: initiate resumable upload session
4. Upload chunks with retry (3 attempts per chunk, exponential backoff)
5. On interruption: persist session URI and byte offset
6. On resume: query upload status, continue from last acknowledged byte
7. On completion: notify user, update local cache, trigger downstream workflows

### Thumbnail Resolution Guide

| Context | Recommended Size | Notes |
|---|---|---|
| List view (phone) | 48x48 or 64x64 | Minimum viable for file type identification |
| Grid view (phone) | 120x120 or 160x160 | Standard thumbnail grid |
| Detail view (phone) | 320x320 | File detail/preview panel |
| List view (tablet) | 80x80 | Larger list items on tablet |
| Grid view (tablet) | 200x200 | More screen real estate |
| Full preview (any) | 800x800 or larger | Before opening external viewer |
