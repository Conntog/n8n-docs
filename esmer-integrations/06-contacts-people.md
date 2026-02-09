# Esmer (ESMO) API Integration Specification: Contacts & People

> **Category:** 06 - Contacts & People
> **Version:** 1.0.0
> **Last Updated:** 2026-02-09
> **Status:** Draft

---

## Table of Contents

- [Cross-Service Comparison Matrix](#cross-service-comparison-matrix)
- [1. Google Contacts (Google People API v1)](#1-google-contacts-google-people-api-v1)
- [2. HubSpot (HubSpot CRM API v3)](#2-hubspot-hubspot-crm-api-v3)
- [3. Pipedrive (Pipedrive REST API)](#3-pipedrive-pipedrive-rest-api)
- [4. Salesforce (Salesforce REST API)](#4-salesforce-salesforce-rest-api)
- [5. Zoho CRM (Zoho CRM API v2)](#5-zoho-crm-zoho-crm-api-v2)
- [6. Freshworks CRM (Freshsales API)](#6-freshworks-crm-freshsales-api)
- [7. Intercom (Intercom API v2)](#7-intercom-intercom-api-v2)
- [8. Copper (Copper REST API)](#8-copper-copper-rest-api)
- [9. Affinity (Affinity API)](#9-affinity-affinity-api)
- [10. Microsoft Dynamics CRM (Dynamics 365 Web API)](#10-microsoft-dynamics-crm-dynamics-365-web-api)

---

## Cross-Service Comparison Matrix

| Feature | Google Contacts | HubSpot | Pipedrive | Salesforce | Zoho CRM | Freshworks CRM | Intercom | Copper | Affinity | MS Dynamics |
|---|---|---|---|---|---|---|---|---|---|---|
| **Auth Method** | OAuth 2.0 | OAuth 2.0 / App Token | OAuth 2.0 / API Token | OAuth 2.0 / JWT | OAuth 2.0 | API Key | API Key (Access Token) | API Key + Email | API Key | OAuth 2.0 |
| **Contacts/People** | CRUD | CRUD + Search | CRUD + Search | CRUD + Upsert | CRUD + Upsert | CRUD | CRUD (Users/Leads) | CRUD (Persons) | CRUD (Persons) | CRUD |
| **Companies/Accounts** | -- | CRUD + Search | CRUD (Organizations) | CRUD + Upsert | CRUD + Upsert | CRUD | CRUD | CRUD | CRUD (Organizations) | CRUD |
| **Deals/Opportunities** | -- | CRUD + Search | CRUD + Search | CRUD + Upsert | CRUD + Upsert | CRUD | -- | CRUD | -- | -- |
| **Leads** | -- | -- | CRUD | CRUD + Upsert | CRUD + Upsert | -- | CRUD | CRUD | -- | -- |
| **Tasks** | -- | -- | -- | CRUD | -- | CRUD | -- | CRUD | -- | -- |
| **Notes** | -- | -- | CRUD | -- | -- | CUD (no read-single) | -- | -- | -- | -- |
| **Activities** | -- | CRUD (Engagements) | CRUD | -- | -- | Read-only (Sales Activity) | -- | -- | -- | -- |
| **Tickets** | -- | CRUD | -- | Cases (CRUD) | -- | -- | -- | -- | -- | -- |
| **Lists** | -- | Add/Remove Contact | -- | -- | -- | -- | -- | -- | CRUD (Lists + Entries) | -- |
| **Webhooks** | -- | Yes (via app) | Yes | Yes (Streaming/PushTopic) | Yes (Notifications) | Yes | Yes | Yes (Trigger) | Yes (Trigger) | Yes (Webhooks) |
| **Rate Limit** | 90 req/min user | 100-200 req/10s | 80 req/2s (API token) | 100,000 req/day | 100 req/min/user | Varies by plan | 1000 req/min | 600 req/10min | 900 req/min | Service Protection |
| **Batch Support** | Yes | Yes | -- | Composite API | Yes | -- | -- | -- | -- | Yes ($batch) |
| **Real-time Sync** | -- | Webhooks | Webhooks | Streaming API | Notifications API | Webhooks | Webhooks | Webhooks | Webhooks | Webhooks |
| **Mobile Priority** | High | High | Medium | High | Medium | Medium | Medium | Low | Low | Medium |

---

## 1. Google Contacts (Google People API v1)

### 1.1 Service Overview

Esmer uses the Google People API to manage the user's personal and organizational contacts stored in Google. Key delegated tasks include:

- Looking up contacts by name, email, or phone number before placing calls or sending messages
- Creating new contacts from business cards, emails, or meeting notes
- Updating contact details (phone numbers, emails, addresses, job titles)
- Syncing contact groups/labels for organizational purposes
- Retrieving contact photos for display in the Esmer contact list UI

### 1.2 Authentication

**Method:** OAuth 2.0 (Authorization Code flow with PKCE recommended for mobile)

**Token Endpoint:** `https://oauth2.googleapis.com/token`
**Authorization Endpoint:** `https://accounts.google.com/o/oauth2/v2/auth`

**Required Scopes:**
| Scope | Purpose |
|---|---|
| `https://www.googleapis.com/auth/contacts` | Full read/write access to contacts |
| `https://www.googleapis.com/auth/contacts.readonly` | Read-only access to contacts |
| `https://www.googleapis.com/auth/contacts.other.readonly` | Read-only access to "Other contacts" |
| `https://www.googleapis.com/auth/directory.readonly` | Read-only access to Google Workspace directory |

**Esmer Recommendation:** Request `contacts` (full read/write) scope. Use PKCE flow for mobile. Store refresh tokens in the device keychain. Access tokens expire after 3600 seconds.

**Token Refresh:**
```
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

client_id=CLIENT_ID&
grant_type=refresh_token&
refresh_token=REFRESH_TOKEN
```

### 1.3 Base URL

```
https://people.googleapis.com/v1
```

### 1.4 Rate Limits

| Quota | Limit |
|---|---|
| Read requests per minute per user | 90 |
| Write requests per minute per user | 60 |
| Read requests per minute per project | 900 |
| Write requests per minute per project | 600 |
| Critical read requests per minute per project | 2700 |

**Esmer Strategy:** Implement per-user request queuing. Cache contact lists locally on-device with a TTL of 5 minutes. Batch read requests where possible using `people.getBatchGet`.

### 1.5 API Endpoints

#### 1.5.1 Contacts (People)

**Create a Contact**

```
POST /v1/people:createContact
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `personFields` | query string | No | Fields to return in the response (e.g., `names,emailAddresses,phoneNumbers`) |

**Request Body:**
```json
{
  "names": [
    {
      "givenName": "Jane",
      "familyName": "Doe"
    }
  ],
  "emailAddresses": [
    {
      "value": "jane.doe@example.com",
      "type": "work"
    }
  ],
  "phoneNumbers": [
    {
      "value": "+1-555-0100",
      "type": "mobile"
    }
  ],
  "organizations": [
    {
      "name": "Acme Corp",
      "title": "VP of Engineering"
    }
  ],
  "addresses": [
    {
      "streetAddress": "123 Main St",
      "city": "San Francisco",
      "region": "CA",
      "postalCode": "94105",
      "country": "US",
      "type": "work"
    }
  ],
  "biographies": [
    {
      "value": "Met at TechCrunch Disrupt 2025"
    }
  ]
}
```

**Response (200 OK):**
```json
{
  "resourceName": "people/c1234567890",
  "etag": "%EgUBAi43PRoEAQIFByIMR2ZGYkVOcjRMdz0=",
  "names": [
    {
      "metadata": {
        "primary": true,
        "source": { "type": "CONTACT", "id": "1234567890" }
      },
      "displayName": "Jane Doe",
      "familyName": "Doe",
      "givenName": "Jane",
      "displayNameLastFirst": "Doe, Jane"
    }
  ],
  "emailAddresses": [
    {
      "metadata": {
        "primary": true,
        "source": { "type": "CONTACT", "id": "1234567890" }
      },
      "value": "jane.doe@example.com",
      "type": "work",
      "formattedType": "Work"
    }
  ],
  "phoneNumbers": [
    {
      "metadata": {
        "primary": true,
        "source": { "type": "CONTACT", "id": "1234567890" }
      },
      "value": "+1-555-0100",
      "type": "mobile",
      "formattedType": "Mobile"
    }
  ]
}
```

**Get a Contact**

```
GET /v1/{resourceName}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `resourceName` | path | Yes | Resource name, e.g., `people/c1234567890` |
| `personFields` | query | Yes | Comma-separated list of fields to return |

**Example:**
```
GET /v1/people/c1234567890?personFields=names,emailAddresses,phoneNumbers,organizations,photos
```

**Response (200 OK):**
```json
{
  "resourceName": "people/c1234567890",
  "etag": "%EgUBAi43PRoEAQIFByIMR2ZGYkVOcjRMdz0=",
  "names": [
    {
      "metadata": { "primary": true, "source": { "type": "CONTACT", "id": "1234567890" } },
      "displayName": "Jane Doe",
      "givenName": "Jane",
      "familyName": "Doe"
    }
  ],
  "photos": [
    {
      "metadata": { "primary": true, "source": { "type": "CONTACT", "id": "1234567890" } },
      "url": "https://lh3.googleusercontent.com/..."
    }
  ]
}
```

**List All Contacts**

```
GET /v1/people/me/connections
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `personFields` | query | Yes | Fields to return |
| `pageSize` | query | No | Max results per page (1-1000, default 100) |
| `pageToken` | query | No | Token for pagination |
| `sortOrder` | query | No | `LAST_MODIFIED_ASCENDING`, `LAST_MODIFIED_DESCENDING`, `FIRST_NAME_ASCENDING`, `LAST_NAME_ASCENDING` |
| `requestSyncToken` | query | No | Set to `true` to receive a `nextSyncToken` for incremental sync |

**Response (200 OK):**
```json
{
  "connections": [
    {
      "resourceName": "people/c1234567890",
      "etag": "...",
      "names": [{ "displayName": "Jane Doe", "givenName": "Jane", "familyName": "Doe" }],
      "emailAddresses": [{ "value": "jane@example.com" }]
    }
  ],
  "nextPageToken": "CAoQAhgB...",
  "totalPeople": 547,
  "totalItems": 547
}
```

**Update a Contact**

```
PATCH /v1/{resourceName}:updateContact
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `resourceName` | path | Yes | Resource name of the contact |
| `updatePersonFields` | query | Yes | Comma-separated list of person fields to update |
| `personFields` | query | No | Fields to return in the response |

**Request Body:**
```json
{
  "etag": "%EgUBAi43PRoEAQIFByIMR2ZGYkVOcjRMdz0=",
  "phoneNumbers": [
    {
      "value": "+1-555-0200",
      "type": "work"
    }
  ]
}
```

**Response (200 OK):** Returns the updated Person resource.

**Delete a Contact**

```
DELETE /v1/{resourceName}:deleteContact
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `resourceName` | path | Yes | Resource name of the contact |

**Response (200 OK):** Empty body on success.

**Batch Get Contacts**

```
GET /v1/people:batchGet
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `resourceNames` | query (repeated) | Yes | Up to 200 resource names |
| `personFields` | query | Yes | Fields to return |

**Response (200 OK):**
```json
{
  "responses": [
    {
      "httpStatusCode": 200,
      "person": {
        "resourceName": "people/c1234567890",
        "names": [{ "displayName": "Jane Doe" }]
      }
    }
  ]
}
```

**Search Contacts**

```
GET /v1/people:searchContacts
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `query` | query | Yes | Search query string (name, email, phone) |
| `readMask` | query | Yes | Fields to return |
| `pageSize` | query | No | Max results (1-30, default 10) |

**Response (200 OK):**
```json
{
  "results": [
    {
      "person": {
        "resourceName": "people/c1234567890",
        "names": [{ "displayName": "Jane Doe" }],
        "emailAddresses": [{ "value": "jane@example.com" }]
      }
    }
  ]
}
```

#### 1.5.2 Contact Groups

**List Contact Groups**

```
GET /v1/contactGroups
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `pageSize` | query | No | Max results (1-1000, default 30) |
| `pageToken` | query | No | Pagination token |
| `groupFields` | query | No | Fields to return (default `groupType,memberCount,metadata,name`) |

**Create Contact Group**

```
POST /v1/contactGroups
```

**Request Body:**
```json
{
  "contactGroup": {
    "name": "VIP Clients"
  }
}
```

**Response (200 OK):**
```json
{
  "resourceName": "contactGroups/abc123",
  "etag": "...",
  "name": "VIP Clients",
  "groupType": "USER_CONTACT_GROUP",
  "memberCount": 0
}
```

**Modify Contact Group Members**

```
POST /v1/{resourceName}/members:modify
```

**Request Body:**
```json
{
  "resourceNamesToAdd": ["people/c111", "people/c222"],
  "resourceNamesToRemove": ["people/c333"]
}
```

**Response (200 OK):**
```json
{
  "notFoundResourceNames": [],
  "canNotRemoveLastContactGroupResourceNames": []
}
```

**Delete Contact Group**

```
DELETE /v1/{resourceName}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `resourceName` | path | Yes | e.g., `contactGroups/abc123` |
| `deleteContacts` | query | No | Also delete contacts in this group (default `false`) |

### 1.6 Webhooks / Real-time

Google People API does not offer webhooks. Esmer implements polling-based sync:

- Use `requestSyncToken=true` on list calls to get a `syncToken`
- On subsequent calls pass `syncToken` to receive only changed contacts
- Poll every 5 minutes in the background; increase to 1 minute when the app is foregrounded
- Fall back to full sync if the sync token expires (HTTP 410 Gone)

### 1.7 Error Handling

| HTTP Code | Error | Description | Esmer Action |
|---|---|---|---|
| 400 | `INVALID_ARGUMENT` | Malformed request or invalid fields | Fix request parameters |
| 401 | `UNAUTHENTICATED` | Token expired or invalid | Refresh access token, retry |
| 403 | `PERMISSION_DENIED` | Missing required scope | Re-prompt user for consent |
| 404 | `NOT_FOUND` | Contact does not exist | Remove from local cache |
| 409 | `ABORTED` | Etag mismatch (concurrent edit) | Re-fetch contact, merge changes, retry |
| 410 | `GONE` | Sync token expired | Perform full sync |
| 429 | `RESOURCE_EXHAUSTED` | Rate limit exceeded | Exponential backoff, retry after `Retry-After` header |
| 500 | `INTERNAL` | Google server error | Retry with exponential backoff (max 3 retries) |
| 503 | `UNAVAILABLE` | Service temporarily unavailable | Retry with exponential backoff |

### 1.8 Mobile-Specific Notes

- **Contact Photos:** Use `photos` field to fetch thumbnail URLs. Cache images on-device with a 24-hour TTL. The URLs require the OAuth token as a cookie to access.
- **Sync Strategy:** Use incremental sync tokens to minimize data transfer on cellular networks. Store contacts in a local SQLite database.
- **Offline Support:** Queue create/update/delete operations when offline. Replay queue on reconnect using etag-based conflict resolution.
- **Deep Links:** Use `resourceName` as the stable identifier to link Esmer contacts to Google Contacts entries.
- **Battery:** Schedule background sync using WorkManager (Android) or BGTaskScheduler (iOS) at 15-minute intervals minimum.

---

## 2. HubSpot (HubSpot CRM API v3)

### 2.1 Service Overview

Esmer uses HubSpot as a full-featured CRM backend. Delegated tasks include:

- Creating and updating contacts when users meet new people or receive business cards
- Managing companies and associating contacts to companies
- Tracking deals through pipeline stages
- Logging engagements (calls, emails, meetings, notes) against contacts
- Managing contact lists for segmentation
- Submitting form data for lead capture
- Managing support tickets

### 2.2 Authentication

**Recommended Method:** OAuth 2.0

**Authorization Endpoint:** `https://app.hubspot.com/oauth/authorize`
**Token Endpoint:** `https://api.hubapi.com/oauth/v1/token`

**Required Scopes:**
| Scope | Purpose |
|---|---|
| `oauth` | Base OAuth scope |
| `crm.objects.contacts.read` | Read contacts |
| `crm.objects.contacts.write` | Write contacts |
| `crm.schemas.contacts.read` | Read contact schemas |
| `crm.objects.companies.read` | Read companies |
| `crm.objects.companies.write` | Write companies |
| `crm.schemas.companies.read` | Read company schemas |
| `crm.objects.deals.read` | Read deals |
| `crm.objects.deals.write` | Write deals |
| `crm.schemas.deals.read` | Read deal schemas |
| `crm.objects.owners.read` | Read owners |
| `crm.lists.write` | Manage contact lists |
| `forms` | Submit forms |
| `tickets` | Manage tickets |

**Alternative Method:** App Token (Private App)
- Generate via HubSpot Settings > Integrations > Private Apps
- Pass as `Authorization: Bearer {app_token}` header
- Simpler setup but less granular permissions

**Esmer Recommendation:** Use OAuth 2.0 for user-facing flows. Access tokens expire after 1800 seconds (30 minutes). Refresh tokens do not expire but are single-use (each refresh returns a new refresh token).

### 2.3 Base URL

```
https://api.hubapi.com
```

### 2.4 Rate Limits

| Tier | Limit |
|---|---|
| OAuth apps | 100 requests per 10 seconds per account |
| Private apps | 200 requests per 10 seconds per account |
| Search endpoints | 4 requests per second |
| Burst limit | 150 requests per 10 seconds |
| Daily limit (free accounts) | 250,000 requests/day |

**Esmer Strategy:** Implement a token-bucket rate limiter per HubSpot account. Use batch APIs for bulk operations. Cache frequently accessed contacts and companies locally.

### 2.5 API Endpoints

#### 2.5.1 Contacts

**Create a Contact**

```
POST /crm/v3/objects/contacts
```

**Headers:**
```
Authorization: Bearer {access_token}
Content-Type: application/json
```

**Request Body:**
```json
{
  "properties": {
    "email": "jane.doe@example.com",
    "firstname": "Jane",
    "lastname": "Doe",
    "phone": "+1-555-0100",
    "company": "Acme Corp",
    "jobtitle": "VP of Engineering",
    "lifecyclestage": "lead",
    "hs_lead_status": "NEW"
  }
}
```

**Response (201 Created):**
```json
{
  "id": "501",
  "properties": {
    "createdate": "2026-02-09T10:00:00.000Z",
    "email": "jane.doe@example.com",
    "firstname": "Jane",
    "hs_object_id": "501",
    "lastmodifieddate": "2026-02-09T10:00:00.000Z",
    "lastname": "Doe"
  },
  "createdAt": "2026-02-09T10:00:00.000Z",
  "updatedAt": "2026-02-09T10:00:00.000Z",
  "archived": false
}
```

**Get a Contact**

```
GET /crm/v3/objects/contacts/{contactId}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `contactId` | path | Yes | Contact ID |
| `properties` | query | No | Comma-separated list of properties to return |
| `associations` | query | No | Comma-separated list of object types to retrieve associations for |

**Response (200 OK):**
```json
{
  "id": "501",
  "properties": {
    "createdate": "2026-02-09T10:00:00.000Z",
    "email": "jane.doe@example.com",
    "firstname": "Jane",
    "lastname": "Doe",
    "phone": "+1-555-0100",
    "company": "Acme Corp",
    "jobtitle": "VP of Engineering",
    "lifecyclestage": "lead",
    "hs_lead_status": "NEW",
    "hs_object_id": "501",
    "lastmodifieddate": "2026-02-09T10:00:00.000Z"
  },
  "createdAt": "2026-02-09T10:00:00.000Z",
  "updatedAt": "2026-02-09T10:00:00.000Z",
  "archived": false
}
```

**Get All Contacts**

```
GET /crm/v3/objects/contacts
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `limit` | query | No | Max results per page (max 100) |
| `after` | query | No | Pagination cursor |
| `properties` | query | No | Properties to return |

**Response (200 OK):**
```json
{
  "results": [
    {
      "id": "501",
      "properties": { "firstname": "Jane", "lastname": "Doe", "email": "jane@example.com" },
      "createdAt": "2026-02-09T10:00:00.000Z",
      "updatedAt": "2026-02-09T10:00:00.000Z",
      "archived": false
    }
  ],
  "paging": {
    "next": {
      "after": "502",
      "link": "https://api.hubapi.com/crm/v3/objects/contacts?after=502"
    }
  }
}
```

**Update a Contact**

```
PATCH /crm/v3/objects/contacts/{contactId}
```

**Request Body:**
```json
{
  "properties": {
    "phone": "+1-555-0200",
    "jobtitle": "CTO"
  }
}
```

**Response (200 OK):** Returns the updated contact object.

**Delete a Contact**

```
DELETE /crm/v3/objects/contacts/{contactId}
```

**Response (204 No Content):** Empty body on success. Archives the contact (soft delete).

**Search Contacts**

```
POST /crm/v3/objects/contacts/search
```

**Request Body:**
```json
{
  "filterGroups": [
    {
      "filters": [
        {
          "propertyName": "email",
          "operator": "CONTAINS_TOKEN",
          "value": "example.com"
        }
      ]
    }
  ],
  "sorts": [
    {
      "propertyName": "createdate",
      "direction": "DESCENDING"
    }
  ],
  "properties": ["firstname", "lastname", "email", "phone"],
  "limit": 10,
  "after": 0
}
```

**Search Operators:** `EQ`, `NEQ`, `LT`, `LTE`, `GT`, `GTE`, `HAS_PROPERTY`, `NOT_HAS_PROPERTY`, `CONTAINS_TOKEN`, `NOT_CONTAINS_TOKEN`, `IN`, `NOT_IN`, `BETWEEN`

**Response (200 OK):**
```json
{
  "total": 3,
  "results": [
    {
      "id": "501",
      "properties": { "firstname": "Jane", "lastname": "Doe", "email": "jane@example.com" }
    }
  ]
}
```

**Get Recently Modified Contacts**

```
GET /crm/v3/objects/contacts?sort=-hs_lastmodifieddate&limit=50
```

#### 2.5.2 Contact Lists

**Add Contact to a List**

```
POST /contacts/v1/lists/{listId}/add
```

**Request Body:**
```json
{
  "vids": [501, 502, 503]
}
```

**Response (200 OK):**
```json
{
  "updated": [501, 502],
  "discarded": [503],
  "invalidVids": [],
  "invalidEmails": []
}
```

**Remove Contact from a List**

```
POST /contacts/v1/lists/{listId}/remove
```

**Request Body:**
```json
{
  "vids": [501]
}
```

#### 2.5.3 Companies

**Create a Company**

```
POST /crm/v3/objects/companies
```

**Request Body:**
```json
{
  "properties": {
    "name": "Acme Corp",
    "domain": "acme.com",
    "industry": "Technology",
    "phone": "+1-555-0000",
    "city": "San Francisco",
    "state": "California",
    "country": "United States",
    "numberofemployees": "500",
    "annualrevenue": "50000000"
  }
}
```

**Response (201 Created):**
```json
{
  "id": "101",
  "properties": {
    "name": "Acme Corp",
    "domain": "acme.com",
    "createdate": "2026-02-09T10:00:00.000Z",
    "hs_object_id": "101",
    "lastmodifieddate": "2026-02-09T10:00:00.000Z"
  },
  "createdAt": "2026-02-09T10:00:00.000Z",
  "updatedAt": "2026-02-09T10:00:00.000Z",
  "archived": false
}
```

**Get a Company**

```
GET /crm/v3/objects/companies/{companyId}
```

**Get All Companies**

```
GET /crm/v3/objects/companies
```

**Update a Company**

```
PATCH /crm/v3/objects/companies/{companyId}
```

**Delete a Company**

```
DELETE /crm/v3/objects/companies/{companyId}
```

**Search Companies by Domain**

```
POST /crm/v3/objects/companies/search
```

**Request Body:**
```json
{
  "filterGroups": [
    {
      "filters": [
        {
          "propertyName": "domain",
          "operator": "EQ",
          "value": "acme.com"
        }
      ]
    }
  ],
  "properties": ["name", "domain", "industry"]
}
```

#### 2.5.4 Deals

**Create a Deal**

```
POST /crm/v3/objects/deals
```

**Request Body:**
```json
{
  "properties": {
    "dealname": "Acme Corp Enterprise License",
    "dealstage": "qualifiedtobuy",
    "pipeline": "default",
    "amount": "150000",
    "closedate": "2026-06-30T00:00:00.000Z",
    "hubspot_owner_id": "12345"
  }
}
```

**Response (201 Created):**
```json
{
  "id": "201",
  "properties": {
    "amount": "150000",
    "closedate": "2026-06-30T00:00:00.000Z",
    "createdate": "2026-02-09T10:00:00.000Z",
    "dealname": "Acme Corp Enterprise License",
    "dealstage": "qualifiedtobuy",
    "hs_object_id": "201",
    "lastmodifieddate": "2026-02-09T10:00:00.000Z",
    "pipeline": "default"
  },
  "createdAt": "2026-02-09T10:00:00.000Z",
  "updatedAt": "2026-02-09T10:00:00.000Z",
  "archived": false
}
```

**Get / Get All / Update / Delete / Search Deals** follow the same pattern as Contacts with endpoint base `/crm/v3/objects/deals`.

#### 2.5.5 Engagements

**Create an Engagement**

```
POST /engagements/v1/engagements
```

**Request Body (Note):**
```json
{
  "engagement": {
    "active": true,
    "type": "NOTE",
    "timestamp": 1707436800000
  },
  "associations": {
    "contactIds": [501],
    "companyIds": [101],
    "dealIds": [201]
  },
  "metadata": {
    "body": "Had a productive call with Jane about the enterprise deal."
  }
}
```

**Engagement Types:** `NOTE`, `EMAIL`, `TASK`, `MEETING`, `CALL`

**Get an Engagement**

```
GET /engagements/v1/engagements/{engagementId}
```

**Get All Engagements**

```
GET /engagements/v1/engagements/paged
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `limit` | query | No | Max results per page (max 250) |
| `offset` | query | No | Pagination offset |

**Delete an Engagement**

```
DELETE /engagements/v1/engagements/{engagementId}
```

#### 2.5.6 Tickets

**Create a Ticket**

```
POST /crm/v3/objects/tickets
```

**Request Body:**
```json
{
  "properties": {
    "hs_pipeline": "0",
    "hs_pipeline_stage": "1",
    "hs_ticket_priority": "HIGH",
    "subject": "Cannot access dashboard"
  }
}
```

**Get / Get All / Update / Delete Tickets** follow the standard CRM v3 pattern at `/crm/v3/objects/tickets`.

#### 2.5.7 Forms

**Get All Form Fields**

```
GET /forms/v2/fields/{formGuid}
```

**Submit Form Data**

```
POST /submissions/v3/integration/submit/{portalId}/{formGuid}
```

**Request Body:**
```json
{
  "fields": [
    { "name": "email", "value": "jane@example.com" },
    { "name": "firstname", "value": "Jane" },
    { "name": "lastname", "value": "Doe" }
  ],
  "context": {
    "pageUri": "https://app.esmer.ai/contact-form",
    "pageName": "Esmer Contact Form"
  }
}
```

### 2.6 Webhooks / Real-time

HubSpot supports webhooks via public apps (not private apps):

**Subscription Types:**
- `contact.creation`, `contact.deletion`, `contact.propertyChange`
- `company.creation`, `company.deletion`, `company.propertyChange`
- `deal.creation`, `deal.deletion`, `deal.propertyChange`

**Webhook Payload:**
```json
[
  {
    "eventId": 1,
    "subscriptionId": 12345,
    "portalId": 67890,
    "appId": 11111,
    "occurredAt": 1707436800000,
    "subscriptionType": "contact.creation",
    "attemptNumber": 0,
    "objectId": 501,
    "changeSource": "CRM",
    "propertyName": null,
    "propertyValue": null
  }
]
```

**Esmer Strategy:** Register webhooks for contact and deal changes. Use the Esmer backend relay to push changes to the mobile app via push notifications or WebSocket.

### 2.7 Error Handling

| HTTP Code | Error Category | Description | Esmer Action |
|---|---|---|---|
| 400 | `VALIDATION_ERROR` | Invalid request body or parameters | Display validation errors to user |
| 401 | `UNAUTHORIZED` | Token expired or invalid | Refresh OAuth token, retry |
| 403 | `FORBIDDEN` | Insufficient scopes or permissions | Re-authenticate with required scopes |
| 404 | `NOT_FOUND` | Object does not exist | Remove from local cache |
| 409 | `CONFLICT` | Duplicate record (e.g., email exists) | Offer to update existing contact |
| 429 | `RATE_LIMIT` | Rate limit exceeded | Exponential backoff; check `Retry-After` header |
| 500 | `INTERNAL_ERROR` | HubSpot server error | Retry with backoff (max 3 attempts) |

**Error Response Format:**
```json
{
  "status": "error",
  "message": "Property values were not valid: [{\"isValid\":false,\"message\":\"Email address jane@example is invalid\",\"error\":\"INVALID_EMAIL\",\"name\":\"email\"}]",
  "correlationId": "abc-def-123",
  "category": "VALIDATION_ERROR"
}
```

### 2.8 Mobile-Specific Notes

- **Quick Contact Capture:** Use the create-or-update endpoint (`POST /crm/v3/objects/contacts` with `idProperty=email`) so duplicate contacts are handled gracefully.
- **Offline Queue:** Queue CRM operations (create contact, update deal stage, log note) locally. Replay on reconnect.
- **Data Volume:** Use the `properties` query parameter to request only needed fields. Avoid fetching all properties for list views.
- **Association Preloading:** For a contact detail view, fetch associations (companies, deals) in a second parallel request.
- **Push Notifications:** Use HubSpot webhooks to relay deal stage changes and new contact assignments to the mobile device.

---

## 3. Pipedrive (Pipedrive REST API)

### 3.1 Service Overview

Esmer uses Pipedrive as a sales-focused CRM for users who prefer pipeline-driven contact and deal management. Delegated tasks include:

- Managing persons (contacts) and organizations
- Creating and progressing deals through pipeline stages
- Logging activities (calls, meetings, tasks) against deals and contacts
- Managing leads before they convert to deals
- Attaching notes and files to deals, persons, and organizations
- Searching across persons, deals, and organizations

### 3.2 Authentication

**Method 1: API Token**
- Found in Pipedrive Settings > Personal Preferences > API
- Pass as query parameter `?api_token={token}` or as `Authorization: Bearer {token}` header

**Method 2: OAuth 2.0**
- Authorization URL: `https://oauth.pipedrive.com/oauth/authorize`
- Token URL: `https://oauth.pipedrive.com/oauth/token`
- Access tokens expire after 3600 seconds (1 hour)
- Refresh tokens expire after 3 hours of inactivity

**Required Scopes (OAuth):**
| Scope | Purpose |
|---|---|
| `contacts:full` | Full access to persons and organizations |
| `contacts:read` | Read-only access to persons and organizations |
| `deals:full` | Full access to deals |
| `deals:read` | Read-only access to deals |
| `activities:full` | Full access to activities |
| `activities:read` | Read-only access to activities |
| `leads:full` | Full access to leads |
| `leads:read` | Read-only access to leads |
| `products:read` | Read-only access to products |
| `products:full` | Full access to products |

**Esmer Recommendation:** Use OAuth 2.0 for production. API token is suitable for development. Request `contacts:full`, `deals:full`, `activities:full`, and `leads:full` scopes for Esmer's feature set.

### 3.3 Base URL

```
https://{company_domain}.pipedrive.com/api/v1
```

For OAuth apps, the company domain is returned in the token response. Alternatively:

```
https://api.pipedrive.com/v1
```

### 3.4 Rate Limits

| Auth Method | Limit |
|---|---|
| API Token | 80 requests per 2 seconds per token |
| OAuth 2.0 | 80 requests per 2 seconds per access token |
| Daily limit | 8,000 requests per day (free), unlimited (paid) |

**Rate Limit Headers:**
```
X-RateLimit-Limit: 80
X-RateLimit-Remaining: 74
X-RateLimit-Reset: 2
```

**Esmer Strategy:** Respect the `X-RateLimit-Remaining` header. When remaining drops below 10, throttle requests. Use exponential backoff on 429 responses.

### 3.5 API Endpoints

#### 3.5.1 Persons (Contacts)

**Create a Person**

```
POST /v1/persons
```

**Request Body:**
```json
{
  "name": "Jane Doe",
  "email": ["jane.doe@example.com"],
  "phone": ["+1-555-0100"],
  "org_id": 101,
  "visible_to": 3,
  "add_time": "2026-02-09 10:00:00",
  "label": 1
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": 501,
    "company_id": 12345,
    "name": "Jane Doe",
    "first_name": "Jane",
    "last_name": "Doe",
    "email": [
      { "value": "jane.doe@example.com", "primary": true, "label": "work" }
    ],
    "phone": [
      { "value": "+1-555-0100", "primary": true, "label": "mobile" }
    ],
    "org_id": {
      "name": "Acme Corp",
      "value": 101
    },
    "add_time": "2026-02-09 10:00:00",
    "update_time": "2026-02-09 10:00:00",
    "visible_to": "3",
    "active_flag": true,
    "open_deals_count": 0,
    "closed_deals_count": 0,
    "activities_count": 0,
    "label": 1
  },
  "related_objects": {}
}
```

**Get a Person**

```
GET /v1/persons/{id}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `id` | path | Yes | Person ID |

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": 501,
    "name": "Jane Doe",
    "first_name": "Jane",
    "last_name": "Doe",
    "email": [{ "value": "jane.doe@example.com", "primary": true, "label": "work" }],
    "phone": [{ "value": "+1-555-0100", "primary": true, "label": "mobile" }],
    "org_id": { "name": "Acme Corp", "value": 101 },
    "open_deals_count": 2,
    "closed_deals_count": 1,
    "activities_count": 5,
    "picture_id": { "pictures": { "128": "https://..." } }
  }
}
```

**Get All Persons**

```
GET /v1/persons
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `start` | query | No | Pagination start (default 0) |
| `limit` | query | No | Items per page (max 500) |
| `sort` | query | No | Field and direction, e.g., `name ASC` |
| `filter_id` | query | No | Filter ID for custom views |

**Response (200 OK):**
```json
{
  "success": true,
  "data": [
    { "id": 501, "name": "Jane Doe", "email": [{"value": "jane@example.com"}] },
    { "id": 502, "name": "John Smith", "email": [{"value": "john@example.com"}] }
  ],
  "additional_data": {
    "pagination": {
      "start": 0,
      "limit": 100,
      "more_items_in_collection": true,
      "next_start": 100
    }
  }
}
```

**Update a Person**

```
PUT /v1/persons/{id}
```

**Request Body:**
```json
{
  "name": "Jane A. Doe",
  "phone": ["+1-555-0100", "+1-555-0200"],
  "label": 2
}
```

**Delete a Person**

```
DELETE /v1/persons/{id}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": { "id": 501 }
}
```

**Search Persons**

```
GET /v1/persons/search
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `term` | query | Yes | Search term (min 2 characters) |
| `fields` | query | No | Fields to search: `name`, `email`, `phone`, `notes`, `custom_fields` |
| `exact_match` | query | No | `true` for exact match |
| `start` | query | No | Pagination start |
| `limit` | query | No | Max results |

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "result_score": 0.95,
        "item": {
          "id": 501,
          "name": "Jane Doe",
          "emails": ["jane.doe@example.com"],
          "phones": ["+1-555-0100"],
          "organization": { "id": 101, "name": "Acme Corp" }
        }
      }
    ]
  }
}
```

#### 3.5.2 Organizations

**Create an Organization**

```
POST /v1/organizations
```

**Request Body:**
```json
{
  "name": "Acme Corp",
  "address": "123 Main St, San Francisco, CA 94105",
  "visible_to": 3
}
```

**Get / Get All / Update / Delete / Search** follow the same patterns as Persons at `/v1/organizations`.

#### 3.5.3 Deals

**Create a Deal**

```
POST /v1/deals
```

**Request Body:**
```json
{
  "title": "Acme Corp Enterprise License",
  "value": 150000,
  "currency": "USD",
  "person_id": 501,
  "org_id": 101,
  "stage_id": 1,
  "pipeline_id": 1,
  "expected_close_date": "2026-06-30",
  "status": "open",
  "visible_to": 3
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": 201,
    "title": "Acme Corp Enterprise License",
    "value": 150000,
    "currency": "USD",
    "status": "open",
    "stage_id": 1,
    "pipeline_id": 1,
    "person_id": { "name": "Jane Doe", "value": 501 },
    "org_id": { "name": "Acme Corp", "value": 101 },
    "expected_close_date": "2026-06-30",
    "add_time": "2026-02-09 10:00:00",
    "update_time": "2026-02-09 10:00:00"
  }
}
```

**Get / Get All / Update / Delete / Duplicate / Search** follow the same pattern at `/v1/deals`.

**Duplicate a Deal**

```
POST /v1/deals/{id}/duplicate
```

**Get Deal Activities**

```
GET /v1/deals/{id}/activities
```

#### 3.5.4 Deal Products

**Add Product to Deal**

```
POST /v1/deals/{id}/products
```

**Request Body:**
```json
{
  "product_id": 301,
  "item_price": 500,
  "quantity": 10,
  "discount_percentage": 5
}
```

**Get All Products in a Deal**

```
GET /v1/deals/{id}/products
```

**Update Product in Deal**

```
PUT /v1/deals/{deal_id}/products/{product_attachment_id}
```

**Remove Product from Deal**

```
DELETE /v1/deals/{deal_id}/products/{product_attachment_id}
```

#### 3.5.5 Activities

**Create an Activity**

```
POST /v1/activities
```

**Request Body:**
```json
{
  "subject": "Follow-up call with Jane",
  "type": "call",
  "due_date": "2026-02-10",
  "due_time": "14:00",
  "duration": "00:30",
  "deal_id": 201,
  "person_id": 501,
  "org_id": 101,
  "note": "Discuss enterprise pricing and timeline.",
  "done": 0
}
```

**Get / Get All / Update / Delete** follow the standard pattern at `/v1/activities`.

#### 3.5.6 Leads

**Create a Lead**

```
POST /v1/leads
```

**Request Body:**
```json
{
  "title": "Potential client from conference",
  "person_id": 501,
  "organization_id": 101,
  "value": {
    "amount": 50000,
    "currency": "USD"
  },
  "expected_close_date": "2026-04-30",
  "label_ids": ["abc-uuid-123"]
}
```

**Get / Get All / Update / Delete** follow the standard pattern at `/v1/leads`.

#### 3.5.7 Notes

**Create a Note**

```
POST /v1/notes
```

**Request Body:**
```json
{
  "content": "Jane mentioned budget approval is expected by end of Q1.",
  "deal_id": 201,
  "person_id": 501,
  "org_id": 101,
  "pinned_to_deal_flag": 1
}
```

**Get / Get All / Update / Delete** follow the standard pattern at `/v1/notes`.

#### 3.5.8 Files

**Create a File**

```
POST /v1/files
```

Content-Type: `multipart/form-data`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `file` | file | Yes | The file to upload |
| `deal_id` | integer | No | Associated deal ID |
| `person_id` | integer | No | Associated person ID |
| `org_id` | integer | No | Associated organization ID |

**Download a File**

```
GET /v1/files/{id}/download
```

**Get / Delete** follow the standard pattern at `/v1/files`.

#### 3.5.9 Products

**Get All Products**

```
GET /v1/products
```

### 3.6 Webhooks / Real-time

Pipedrive supports webhooks for real-time notifications:

**Create a Webhook**

```
POST /v1/webhooks
```

**Request Body:**
```json
{
  "subscription_url": "https://api.esmer.ai/webhooks/pipedrive",
  "event_action": "updated",
  "event_object": "deal"
}
```

**Event Objects:** `activity`, `deal`, `note`, `organization`, `person`, `pipeline`, `product`, `stage`, `user`

**Event Actions:** `added`, `updated`, `deleted`, `merged`

**Webhook Payload:**
```json
{
  "v": 1,
  "matches_filters": null,
  "meta": {
    "action": "updated",
    "object": "deal",
    "id": 201,
    "company_id": 12345,
    "user_id": 1,
    "timestamp": 1707436800
  },
  "current": {
    "id": 201,
    "title": "Acme Corp Enterprise License",
    "value": 150000,
    "status": "open",
    "stage_id": 2
  },
  "previous": {
    "stage_id": 1
  },
  "event": "updated.deal",
  "retry": 0
}
```

### 3.7 Error Handling

| HTTP Code | Meaning | Esmer Action |
|---|---|---|
| 400 | Bad request / validation error | Display error details to user |
| 401 | Unauthorized (invalid/expired token) | Refresh token or re-authenticate |
| 402 | Payment required (plan limits) | Notify user to upgrade plan |
| 403 | Forbidden (insufficient permissions) | Request additional scopes |
| 404 | Resource not found | Remove from local cache |
| 410 | Gone (resource permanently deleted) | Remove from local cache |
| 429 | Rate limit exceeded | Backoff based on `X-RateLimit-Reset` header |
| 500 | Internal server error | Retry with exponential backoff |

**Standard Error Response:**
```json
{
  "success": false,
  "error": "Person not found",
  "error_info": "Please check the input parameters.",
  "data": null
}
```

### 3.8 Mobile-Specific Notes

- **Person Search:** Use the `/v1/persons/search` endpoint for quick contact lookup from the mobile search bar. The `result_score` field helps rank results.
- **Deal Pipeline View:** Fetch pipeline stages via `GET /v1/stages` then overlay deals. Cache stage definitions locally as they rarely change.
- **Activity Reminders:** Sync upcoming activities to on-device notifications. Poll `/v1/activities?type=call&done=0&start_date=today` periodically.
- **Offline Mode:** Queue deal stage changes and new notes for replay. Use `update_time` to detect conflicts on sync.
- **Compact Payloads:** Pipedrive returns related objects inline. Use custom field mapping to reduce payload parsing overhead on mobile.

---

## 4. Salesforce (Salesforce REST API)

### 4.1 Service Overview

Esmer integrates with Salesforce as an enterprise CRM for managing the full customer lifecycle. Delegated tasks include:

- Managing contacts, accounts, leads, and opportunities (full CRUD + upsert)
- Logging tasks and activities against records
- Attaching files and documents
- Managing cases for customer support
- Executing SOQL queries for custom data retrieval
- Working with custom objects
- Invoking Salesforce Flows from the mobile app

### 4.2 Authentication

**Method 1: OAuth 2.0 (User-Agent or Web Server flow)**
- Authorization URL: `https://login.salesforce.com/services/oauth2/authorize` (Production) or `https://test.salesforce.com/services/oauth2/authorize` (Sandbox)
- Token URL: `https://login.salesforce.com/services/oauth2/token`

**Method 2: JWT Bearer Token Flow**
- Requires a Connected App with a digital certificate
- Generates access tokens without user interaction (server-to-server)
- Private key in PEM format required

**Required Scopes:**
| Scope | Purpose |
|---|---|
| `full` | Full access to all data |
| `refresh_token, offline_access` | Ability to refresh tokens |
| `api` | Access REST API |

**Esmer Recommendation:** Use OAuth 2.0 Web Server flow with PKCE for mobile. Store the instance URL from the token response -- all subsequent API calls must use this instance URL. Access tokens expire based on Salesforce session timeout settings (default 2 hours).

**Token Response:**
```json
{
  "access_token": "00D...",
  "instance_url": "https://yourInstance.salesforce.com",
  "id": "https://login.salesforce.com/id/00Dxx/005xx",
  "token_type": "Bearer",
  "issued_at": "1707436800000",
  "signature": "...",
  "refresh_token": "5Aep..."
}
```

### 4.3 Base URL

```
https://{instance_url}/services/data/v59.0
```

The `instance_url` is returned in the OAuth token response (e.g., `https://na1.salesforce.com` or `https://mycompany.my.salesforce.com`).

### 4.4 Rate Limits

| Limit Type | Value |
|---|---|
| API requests per 24-hour period | Based on edition: Enterprise = 100,000; Unlimited = 500,000 |
| Concurrent API request limit | 25 per user |
| Concurrent long-running requests | 10 per org |
| Batch/Bulk API daily batches | 15,000 per 24 hours |
| SOQL query row limit | 50,000 rows per query |
| API request size limit | 6 MB per request |

**Rate Limit Headers:**
```
Sforce-Limit-Info: api-usage=45/100000
```

**Esmer Strategy:** Monitor `Sforce-Limit-Info` header on every response. Cache account/contact data aggressively. Use Composite API for multi-step operations to reduce request count. Use Bulk API 2.0 for large data syncs.

### 4.5 API Endpoints

#### 4.5.1 Contacts

**Create a Contact**

```
POST /services/data/v59.0/sobjects/Contact
```

**Request Body:**
```json
{
  "FirstName": "Jane",
  "LastName": "Doe",
  "Email": "jane.doe@example.com",
  "Phone": "+1-555-0100",
  "MobilePhone": "+1-555-0101",
  "Title": "VP of Engineering",
  "Department": "Engineering",
  "AccountId": "001xx000003DGbYAAW",
  "MailingStreet": "123 Main St",
  "MailingCity": "San Francisco",
  "MailingState": "CA",
  "MailingPostalCode": "94105",
  "MailingCountry": "US",
  "Description": "Met at TechCrunch Disrupt 2025"
}
```

**Response (201 Created):**
```json
{
  "id": "003xx000004TmiQAAS",
  "success": true,
  "errors": []
}
```

**Get a Contact**

```
GET /services/data/v59.0/sobjects/Contact/{id}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `id` | path | Yes | Salesforce record ID (18-character) |
| `fields` | query | No | Comma-separated field names to return |

**Response (200 OK):**
```json
{
  "attributes": {
    "type": "Contact",
    "url": "/services/data/v59.0/sobjects/Contact/003xx000004TmiQAAS"
  },
  "Id": "003xx000004TmiQAAS",
  "FirstName": "Jane",
  "LastName": "Doe",
  "Email": "jane.doe@example.com",
  "Phone": "+1-555-0100",
  "Title": "VP of Engineering",
  "AccountId": "001xx000003DGbYAAW",
  "Account": {
    "attributes": { "type": "Account" },
    "Name": "Acme Corp"
  },
  "CreatedDate": "2026-02-09T10:00:00.000+0000",
  "LastModifiedDate": "2026-02-09T10:00:00.000+0000"
}
```

**Get All Contacts (via SOQL)**

```
GET /services/data/v59.0/query/?q=SELECT+Id,FirstName,LastName,Email,Phone,Title,AccountId+FROM+Contact+ORDER+BY+LastModifiedDate+DESC+LIMIT+100
```

**Response (200 OK):**
```json
{
  "totalSize": 100,
  "done": true,
  "records": [
    {
      "attributes": { "type": "Contact", "url": "/services/data/v59.0/sobjects/Contact/003xx..." },
      "Id": "003xx000004TmiQAAS",
      "FirstName": "Jane",
      "LastName": "Doe",
      "Email": "jane.doe@example.com"
    }
  ],
  "nextRecordsUrl": "/services/data/v59.0/query/01gxx000000MYuq-2000"
}
```

**Update a Contact**

```
PATCH /services/data/v59.0/sobjects/Contact/{id}
```

**Request Body:**
```json
{
  "Phone": "+1-555-0200",
  "Title": "CTO"
}
```

**Response (204 No Content):** Empty body on success.

**Upsert a Contact**

```
PATCH /services/data/v59.0/sobjects/Contact/{externalIdField}/{externalIdValue}
```

Example using Email as external ID:
```
PATCH /services/data/v59.0/sobjects/Contact/Email/jane.doe@example.com
```

**Request Body:**
```json
{
  "FirstName": "Jane",
  "LastName": "Doe",
  "Phone": "+1-555-0100"
}
```

**Response:** 201 (created) or 204 (updated).

**Delete a Contact**

```
DELETE /services/data/v59.0/sobjects/Contact/{id}
```

**Response (204 No Content):** Empty body on success.

**Get Contact Metadata**

```
GET /services/data/v59.0/sobjects/Contact/describe
```

Returns full schema including all fields, picklist values, and relationships.

**Add Note to Contact**

```
POST /services/data/v59.0/sobjects/Note
```

**Request Body:**
```json
{
  "ParentId": "003xx000004TmiQAAS",
  "Title": "Meeting Notes",
  "Body": "Discussed enterprise pricing. Jane will present to her board next week."
}
```

**Add Contact to Campaign**

```
POST /services/data/v59.0/sobjects/CampaignMember
```

**Request Body:**
```json
{
  "CampaignId": "701xx000001ABCAAA",
  "ContactId": "003xx000004TmiQAAS",
  "Status": "Sent"
}
```

#### 4.5.2 Accounts

**Create an Account**

```
POST /services/data/v59.0/sobjects/Account
```

**Request Body:**
```json
{
  "Name": "Acme Corp",
  "Industry": "Technology",
  "Phone": "+1-555-0000",
  "Website": "https://acme.com",
  "BillingStreet": "123 Main St",
  "BillingCity": "San Francisco",
  "BillingState": "CA",
  "BillingPostalCode": "94105",
  "BillingCountry": "US",
  "NumberOfEmployees": 500,
  "AnnualRevenue": 50000000,
  "Type": "Customer",
  "Description": "Enterprise technology company"
}
```

**Get / Get All (SOQL) / Update / Upsert / Delete / Describe** follow the same patterns as Contact at `/services/data/v59.0/sobjects/Account`.

#### 4.5.3 Leads

**Create a Lead**

```
POST /services/data/v59.0/sobjects/Lead
```

**Request Body:**
```json
{
  "FirstName": "Alex",
  "LastName": "Johnson",
  "Email": "alex@startup.io",
  "Phone": "+1-555-0300",
  "Company": "StartupIO",
  "Title": "Founder",
  "LeadSource": "Trade Show",
  "Status": "Open - Not Contacted",
  "Industry": "Technology"
}
```

**Get / Get All / Update / Upsert / Delete / Describe** follow the same patterns at `/services/data/v59.0/sobjects/Lead`.

#### 4.5.4 Opportunities

**Create an Opportunity**

```
POST /services/data/v59.0/sobjects/Opportunity
```

**Request Body:**
```json
{
  "Name": "Acme Corp - Enterprise License",
  "AccountId": "001xx000003DGbYAAW",
  "Amount": 150000,
  "CloseDate": "2026-06-30",
  "StageName": "Prospecting",
  "Probability": 20,
  "Type": "New Business",
  "LeadSource": "Trade Show",
  "Description": "Enterprise license deal from TechCrunch conference"
}
```

**Get / Get All / Update / Upsert / Delete / Describe** follow the same patterns at `/services/data/v59.0/sobjects/Opportunity`.

#### 4.5.5 Tasks

**Create a Task**

```
POST /services/data/v59.0/sobjects/Task
```

**Request Body:**
```json
{
  "Subject": "Follow up with Jane Doe",
  "WhoId": "003xx000004TmiQAAS",
  "WhatId": "006xx000001XYZAAA",
  "ActivityDate": "2026-02-15",
  "Priority": "High",
  "Status": "Not Started",
  "Description": "Send enterprise pricing document and schedule demo."
}
```

**Get / Get All / Update / Delete / Describe** follow the same patterns at `/services/data/v59.0/sobjects/Task`.

#### 4.5.6 Cases

**Create a Case**

```
POST /services/data/v59.0/sobjects/Case
```

**Request Body:**
```json
{
  "ContactId": "003xx000004TmiQAAS",
  "AccountId": "001xx000003DGbYAAW",
  "Subject": "Cannot access dashboard",
  "Description": "User reports dashboard loading error since Feb 8.",
  "Priority": "High",
  "Status": "New",
  "Origin": "Phone",
  "Type": "Problem"
}
```

**Add Comment to Case**

```
POST /services/data/v59.0/sobjects/CaseComment
```

**Request Body:**
```json
{
  "ParentId": "500xx000001ABCAAA",
  "CommentBody": "Investigated the issue. Root cause identified. Fix deploying tonight."
}
```

#### 4.5.7 Attachments

**Create an Attachment**

```
POST /services/data/v59.0/sobjects/Attachment
```

**Request Body:**
```json
{
  "ParentId": "003xx000004TmiQAAS",
  "Name": "proposal.pdf",
  "Body": "<base64-encoded-content>",
  "ContentType": "application/pdf"
}
```

**Get / Update / Delete** follow the standard sObject pattern.

#### 4.5.8 Custom Objects

**Create Custom Object Record**

```
POST /services/data/v59.0/sobjects/{CustomObject__c}
```

**Request Body (example):**
```json
{
  "Name": "Custom Record 001",
  "Custom_Field__c": "value",
  "Lookup_Field__c": "003xx000004TmiQAAS"
}
```

#### 4.5.9 Documents

**Upload a Document (ContentVersion)**

```
POST /services/data/v59.0/sobjects/ContentVersion
```

**Request Body (multipart):**
```
Content-Type: multipart/form-data

--boundary
Content-Disposition: form-data; name="entity_content"
Content-Type: application/json

{"Title":"Pricing Document","PathOnClient":"pricing.pdf"}
--boundary
Content-Disposition: form-data; name="VersionData"; filename="pricing.pdf"
Content-Type: application/pdf

<binary data>
--boundary--
```

#### 4.5.10 Flows

**Get All Flows**

```
GET /services/data/v59.0/actions/custom/flow
```

**Invoke a Flow**

```
POST /services/data/v59.0/actions/custom/flow/{flowApiName}
```

**Request Body:**
```json
{
  "inputs": [
    {
      "contactId": "003xx000004TmiQAAS",
      "actionType": "sendWelcomeEmail"
    }
  ]
}
```

#### 4.5.11 SOQL Search

**Execute a SOQL Query**

```
GET /services/data/v59.0/query/?q={soql_query}
```

**Example:**
```
GET /services/data/v59.0/query/?q=SELECT+Id,Name,Email+FROM+Contact+WHERE+Account.Name='Acme Corp'+AND+LastModifiedDate>2026-01-01T00:00:00Z
```

**Response (200 OK):**
```json
{
  "totalSize": 5,
  "done": true,
  "records": [
    {
      "attributes": { "type": "Contact" },
      "Id": "003xx...",
      "Name": "Jane Doe",
      "Email": "jane@example.com"
    }
  ]
}
```

**Paginated Query (query more):**
```
GET /services/data/v59.0/query/{queryLocator}
```

#### 4.5.12 Users

**Get a User**

```
GET /services/data/v59.0/sobjects/User/{id}
```

**Get All Users (SOQL)**

```
GET /services/data/v59.0/query/?q=SELECT+Id,Name,Email,IsActive+FROM+User+WHERE+IsActive=true
```

#### 4.5.13 Composite API (Batch Operations)

**Composite Request (up to 25 sub-requests)**

```
POST /services/data/v59.0/composite
```

**Request Body:**
```json
{
  "allOrNone": true,
  "compositeRequest": [
    {
      "method": "POST",
      "url": "/services/data/v59.0/sobjects/Account",
      "referenceId": "newAccount",
      "body": { "Name": "New Corp" }
    },
    {
      "method": "POST",
      "url": "/services/data/v59.0/sobjects/Contact",
      "referenceId": "newContact",
      "body": {
        "LastName": "Doe",
        "AccountId": "@{newAccount.id}"
      }
    }
  ]
}
```

### 4.6 Webhooks / Real-time

Salesforce offers multiple real-time mechanisms:

**Platform Events:**
- Publish: `POST /services/data/v59.0/sobjects/{PlatformEventName__e}`
- Subscribe: Use CometD (Bayeux protocol) at `/cometd/59.0`

**Outbound Messages (Workflow Rules):**
- SOAP-based, configured in Salesforce Setup
- Sends XML payloads to registered endpoints

**Change Data Capture (CDC):**
- Subscribe to `/data/ChangeEvents` channel via CometD
- Receives create, update, delete, and undelete notifications

**Esmer Strategy:** Use Platform Events for mobile push relay. Subscribe via the Esmer backend CometD client and forward to mobile via push notifications.

### 4.7 Error Handling

| HTTP Code | Error Code | Description | Esmer Action |
|---|---|---|---|
| 400 | `MALFORMED_QUERY` | Invalid SOQL | Fix query syntax |
| 400 | `REQUIRED_FIELD_MISSING` | Missing required field | Prompt user for missing data |
| 400 | `FIELD_INTEGRITY_EXCEPTION` | Invalid field value | Validate data before submission |
| 400 | `DUPLICATE_VALUE` | Duplicate external ID | Use upsert instead |
| 401 | `INVALID_SESSION_ID` | Token expired | Refresh token, retry |
| 403 | `INSUFFICIENT_ACCESS` | Missing permissions | Notify user/admin |
| 404 | `NOT_FOUND` | Record does not exist | Remove from local cache |
| 429 | `REQUEST_LIMIT_EXCEEDED` | API limit exceeded | Backoff; check `Sforce-Limit-Info` |
| 500 | `UNKNOWN_EXCEPTION` | Server error | Retry with exponential backoff |

**Error Response Format:**
```json
[
  {
    "message": "Required fields are missing: [LastName]",
    "errorCode": "REQUIRED_FIELD_MISSING",
    "fields": ["LastName"]
  }
]
```

### 4.8 Mobile-Specific Notes

- **Instance URL Caching:** Cache the `instance_url` from auth. All API calls must use this URL, not `login.salesforce.com`.
- **SOQL Optimization:** Use `SELECT` with explicit fields (never `SELECT *`). Add `LIMIT` to all queries. Use `WHERE LastModifiedDate > :lastSync` for incremental sync.
- **Composite API:** Batch related operations (create account + create contact + create opportunity) into a single Composite request to reduce round trips on mobile.
- **Field-Level Security:** Salesforce may hide fields based on user profile. Handle missing fields gracefully in the UI.
- **Offline Operations:** Queue CRUD operations with timestamps. On reconnect, check `LastModifiedDate` before applying updates to detect conflicts.
- **File Uploads:** Use chunked upload for large files on mobile. ContentVersion API supports up to 2 GB but use multipart for files under 37.5 MB.
- **Custom Objects:** Dynamically discover custom objects via `describe` endpoint. Cache schema locally with a 24-hour TTL.

---

## 5. Zoho CRM (Zoho CRM API v2)

### 5.1 Service Overview

Esmer uses Zoho CRM for users in the Zoho ecosystem who need contact, lead, deal, and account management. Delegated tasks include:

- Full CRUD and upsert operations on contacts, accounts, leads, and deals
- Managing invoices, quotes, sales orders, purchase orders, and vendor records
- Product catalog management
- Retrieving field metadata for dynamic form generation

### 5.2 Authentication

**Method:** OAuth 2.0

**Authorization Endpoint (region-specific):**
| Region | Authorization URL |
|---|---|
| US | `https://accounts.zoho.com/oauth/v2/auth` |
| EU | `https://accounts.zoho.eu/oauth/v2/auth` |
| IN | `https://accounts.zoho.in/oauth/v2/auth` |
| AU | `https://accounts.zoho.com.au/oauth/v2/auth` |
| CN | `https://accounts.zoho.com.cn/oauth/v2/auth` |

**Token Endpoint (region-specific):**
| Region | Token URL |
|---|---|
| US | `https://accounts.zoho.com/oauth/v2/token` |
| EU | `https://accounts.zoho.eu/oauth/v2/token` |
| IN | `https://accounts.zoho.in/oauth/v2/token` |
| AU | `https://accounts.zoho.com.au/oauth/v2/token` |
| CN | `https://accounts.zoho.com.cn/oauth/v2/token` |

**Required Scopes:**
| Scope | Purpose |
|---|---|
| `ZohoCRM.modules.ALL` | Access all modules |
| `ZohoCRM.modules.contacts.ALL` | Full access to contacts |
| `ZohoCRM.modules.leads.ALL` | Full access to leads |
| `ZohoCRM.modules.deals.ALL` | Full access to deals |
| `ZohoCRM.modules.accounts.ALL` | Full access to accounts |
| `ZohoCRM.settings.ALL` | Access settings and metadata |

**Registration:** Register a Server-based Application at `https://api-console.zoho.com/`. Use the n8n/Esmer OAuth callback URL as the Authorized Redirect URI.

**Esmer Recommendation:** Detect the user's Zoho region during setup and store the correct token/API URLs. Access tokens expire after 3600 seconds. Refresh tokens are long-lived but single-use.

### 5.3 Base URL

```
https://www.zohoapis.com/crm/v2
```

Region-specific alternatives:
| Region | Base URL |
|---|---|
| US | `https://www.zohoapis.com/crm/v2` |
| EU | `https://www.zohoapis.eu/crm/v2` |
| IN | `https://www.zohoapis.in/crm/v2` |
| AU | `https://www.zohoapis.com.au/crm/v2` |
| CN | `https://www.zohoapis.com.cn/crm/v2` |

### 5.4 Rate Limits

| Limit Type | Value |
|---|---|
| API credits per day | Based on edition (Free: 5,000; Standard: 100,000; Professional: 250,000; Enterprise: 500,000) |
| Requests per minute per user | 100 |
| Concurrent requests per user | 25 |
| Batch records per request | 100 |
| Search API calls per day | 10,000 |

**Esmer Strategy:** Cache module metadata. Use batch operations for bulk creates/updates. Respect per-minute limits with request queuing.

### 5.5 API Endpoints

#### 5.5.1 Contacts

**Create a Contact**

```
POST /crm/v2/Contacts
```

**Headers:**
```
Authorization: Zoho-oauthtoken {access_token}
Content-Type: application/json
```

**Request Body:**
```json
{
  "data": [
    {
      "First_Name": "Jane",
      "Last_Name": "Doe",
      "Email": "jane.doe@example.com",
      "Phone": "+1-555-0100",
      "Mobile": "+1-555-0101",
      "Title": "VP of Engineering",
      "Department": "Engineering",
      "Account_Name": {
        "id": "4150868000000307001"
      },
      "Mailing_Street": "123 Main St",
      "Mailing_City": "San Francisco",
      "Mailing_State": "CA",
      "Mailing_Zip": "94105",
      "Mailing_Country": "US",
      "Description": "Met at TechCrunch Disrupt"
    }
  ],
  "trigger": ["workflow"]
}
```

**Response (201 Created):**
```json
{
  "data": [
    {
      "code": "SUCCESS",
      "details": {
        "Modified_Time": "2026-02-09T10:00:00+00:00",
        "Modified_By": { "name": "User", "id": "4150868000000225013" },
        "Created_Time": "2026-02-09T10:00:00+00:00",
        "id": "4150868000000500001",
        "Created_By": { "name": "User", "id": "4150868000000225013" }
      },
      "message": "record added",
      "status": "success"
    }
  ]
}
```

**Get a Contact**

```
GET /crm/v2/Contacts/{record_id}
```

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": "4150868000000500001",
      "First_Name": "Jane",
      "Last_Name": "Doe",
      "Email": "jane.doe@example.com",
      "Phone": "+1-555-0100",
      "Account_Name": {
        "name": "Acme Corp",
        "id": "4150868000000307001"
      },
      "Owner": {
        "name": "User",
        "id": "4150868000000225013"
      },
      "Created_Time": "2026-02-09T10:00:00+00:00",
      "Modified_Time": "2026-02-09T10:00:00+00:00"
    }
  ]
}
```

**Get All Contacts**

```
GET /crm/v2/Contacts
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `fields` | query | No | Comma-separated field API names |
| `sort_order` | query | No | `asc` or `desc` |
| `sort_by` | query | No | Field API name to sort by |
| `page` | query | No | Page number (starts at 1) |
| `per_page` | query | No | Records per page (max 200, default 200) |
| `modified_since` | header | No | ISO 8601 timestamp for incremental sync |

**Update a Contact**

```
PUT /crm/v2/Contacts/{record_id}
```

**Request Body:**
```json
{
  "data": [
    {
      "id": "4150868000000500001",
      "Phone": "+1-555-0200",
      "Title": "CTO"
    }
  ]
}
```

**Upsert a Contact**

```
POST /crm/v2/Contacts/upsert
```

**Request Body:**
```json
{
  "data": [
    {
      "Email": "jane.doe@example.com",
      "First_Name": "Jane",
      "Last_Name": "Doe",
      "Phone": "+1-555-0200"
    }
  ],
  "duplicate_check_fields": ["Email"]
}
```

**Delete a Contact**

```
DELETE /crm/v2/Contacts?ids={record_id1},{record_id2}
```

**Response (200 OK):**
```json
{
  "data": [
    {
      "code": "SUCCESS",
      "details": { "id": "4150868000000500001" },
      "message": "record deleted",
      "status": "success"
    }
  ]
}
```

#### 5.5.2 Accounts

**Create / Get / Get All / Update / Upsert / Delete** follow the same patterns at `/crm/v2/Accounts`.

**Create an Account**

```
POST /crm/v2/Accounts
```

**Request Body:**
```json
{
  "data": [
    {
      "Account_Name": "Acme Corp",
      "Industry": "Technology",
      "Phone": "+1-555-0000",
      "Website": "https://acme.com",
      "Billing_Street": "123 Main St",
      "Billing_City": "San Francisco",
      "Billing_State": "CA",
      "Billing_Code": "94105",
      "Billing_Country": "US",
      "Employees": 500,
      "Annual_Revenue": 50000000
    }
  ]
}
```

#### 5.5.3 Deals

**Create a Deal**

```
POST /crm/v2/Deals
```

**Request Body:**
```json
{
  "data": [
    {
      "Deal_Name": "Acme Corp Enterprise License",
      "Stage": "Qualification",
      "Amount": 150000,
      "Closing_Date": "2026-06-30",
      "Account_Name": { "id": "4150868000000307001" },
      "Contact_Name": { "id": "4150868000000500001" },
      "Pipeline": "Standard"
    }
  ]
}
```

**Get / Get All / Update / Upsert / Delete** follow the same pattern at `/crm/v2/Deals`.

#### 5.5.4 Leads

**Create a Lead**

```
POST /crm/v2/Leads
```

**Request Body:**
```json
{
  "data": [
    {
      "First_Name": "Alex",
      "Last_Name": "Johnson",
      "Email": "alex@startup.io",
      "Phone": "+1-555-0300",
      "Company": "StartupIO",
      "Title": "Founder",
      "Lead_Source": "Trade Show",
      "Lead_Status": "Not Contacted",
      "Industry": "Technology"
    }
  ]
}
```

**Get Lead Fields**

```
GET /crm/v2/settings/fields?module=Leads
```

**Get / Get All / Update / Upsert / Delete** follow the standard module pattern at `/crm/v2/Leads`.

#### 5.5.5 Invoices

**Create / Get / Get All / Update / Upsert / Delete** at `/crm/v2/Invoices`.

#### 5.5.6 Products

**Create / Get / Get All / Update / Upsert / Delete** at `/crm/v2/Products`.

#### 5.5.7 Quotes

**Create / Get / Get All / Update / Upsert / Delete** at `/crm/v2/Quotes`.

#### 5.5.8 Sales Orders

**Create / Get / Get All / Update / Upsert / Delete** at `/crm/v2/Sales_Orders`.

#### 5.5.9 Purchase Orders

**Create / Get / Get All / Update / Upsert / Delete** at `/crm/v2/Purchase_Orders`.

#### 5.5.10 Vendors

**Create / Get / Get All / Update / Upsert / Delete** at `/crm/v2/Vendors`.

### 5.6 Webhooks / Real-time

Zoho CRM supports notifications via the Notification API:

**Enable Notifications**

```
POST /crm/v2/actions/watch
```

**Request Body:**
```json
{
  "watch": [
    {
      "channel_id": "1000001",
      "events": ["Contacts.create", "Contacts.edit", "Contacts.delete"],
      "channel_expiry": "2026-03-09T10:00:00+00:00",
      "token": "esmer_verification_token",
      "notify_url": "https://api.esmer.ai/webhooks/zoho"
    }
  ]
}
```

**Supported Events:** `{Module}.create`, `{Module}.edit`, `{Module}.delete`, `{Module}.all`

**Notification Payload:**
```json
{
  "query_map": {
    "channel_id": "1000001",
    "module": "Contacts",
    "resource_uri": "/crm/v2/Contacts",
    "ids": ["4150868000000500001"],
    "operation": "update",
    "token": "esmer_verification_token",
    "channel_expiry": "2026-03-09T10:00:00+00:00"
  }
}
```

**Esmer Strategy:** Re-register notifications before expiry (max 24-hour channel expiry). Use the notification to trigger a targeted GET request for changed records.

### 5.7 Error Handling

| HTTP Code | Error Code | Description | Esmer Action |
|---|---|---|---|
| 200 | `INVALID_DATA` | Validation failed | Display field-level errors |
| 200 | `DUPLICATE_DATA` | Duplicate record | Offer upsert or merge |
| 200 | `MANDATORY_NOT_FOUND` | Required field missing | Prompt user for field |
| 400 | `BAD_REQUEST` | Malformed request | Fix request format |
| 401 | `INVALID_TOKEN` | Token expired | Refresh token, retry |
| 401 | `AUTHENTICATION_FAILURE` | Invalid credentials | Re-authenticate |
| 403 | `NO_PERMISSION` | Insufficient CRM permissions | Notify user/admin |
| 404 | `INVALID_MODULE` | Module does not exist | Check module name |
| 429 | `API_COUNT_EXCEEDED` | Rate limit hit | Backoff; retry after limit resets |

**Error Response Format:**
```json
{
  "data": [
    {
      "code": "MANDATORY_NOT_FOUND",
      "details": { "api_name": "Last_Name" },
      "message": "required field not found",
      "status": "error"
    }
  ]
}
```

### 5.8 Mobile-Specific Notes

- **Multi-Region:** Detect and persist the user's Zoho region during initial OAuth setup. All subsequent API calls must use the region-specific base URL.
- **Batch Operations:** Zoho supports up to 100 records per create/update request. Batch offline-queued operations into a single API call on reconnect.
- **Modified-Since Header:** Use the `If-Modified-Since` header on GET requests for efficient incremental sync.
- **Field Metadata:** Cache the `/settings/fields` response for each module. Use it to render dynamic forms on the mobile app. Refresh metadata weekly.
- **Upsert Strategy:** Use upsert with `duplicate_check_fields` to avoid duplicate creation from offline queues.

---

## 6. Freshworks CRM (Freshsales API)

### 6.1 Service Overview

Esmer uses Freshworks CRM (Freshsales) for users who need a lightweight CRM with built-in phone and email. Delegated tasks include:

- Managing contacts with lifecycle tracking
- Managing accounts (companies)
- Creating and tracking deals through sales pipelines
- Scheduling appointments and tasks
- Logging notes against records
- Viewing sales activities

### 6.2 Authentication

**Method:** API Key

**Header Format:**
```
Authorization: Token token={api_key}
```

**Obtaining the API Key:**
1. Log in to Freshworks CRM
2. Go to Settings > API Settings
3. Copy the API key

**Domain Configuration:** The API requires the user's Freshworks subdomain (e.g., `n8n` for `https://n8n.myfreshworks.com`).

**Esmer Recommendation:** Store the API key securely in the device keychain. API keys do not expire but can be regenerated by the user, which invalidates the old key.

### 6.3 Base URL

```
https://{domain}.myfreshworks.com/crm/sales/api
```

Where `{domain}` is the user's Freshworks CRM subdomain.

### 6.4 Rate Limits

| Plan | Limit |
|---|---|
| Free | 250 requests per hour |
| Growth | 1,000 requests per hour |
| Pro | 3,000 requests per hour |
| Enterprise | 5,000 requests per hour |

**Rate Limit Headers:**
```
X-RateLimit-Total: 3000
X-RateLimit-Remaining: 2950
X-RateLimit-Used-CurrentRequest: 1
```

**Esmer Strategy:** Track remaining quota via headers. Throttle proactively when below 10% remaining. Cache frequently accessed contacts.

### 6.5 API Endpoints

#### 6.5.1 Contacts

**Create a Contact**

```
POST /api/contacts
```

**Request Body:**
```json
{
  "contact": {
    "first_name": "Jane",
    "last_name": "Doe",
    "email": "jane.doe@example.com",
    "mobile_number": "+1-555-0100",
    "work_number": "+1-555-0101",
    "job_title": "VP of Engineering",
    "sales_accounts": [
      { "id": 101, "is_primary": true }
    ],
    "address": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "zipcode": "94105",
    "country": "US",
    "lifecycle_stage_id": 71010794467,
    "lead_source_id": 71010794468
  }
}
```

**Response (201 Created):**
```json
{
  "contact": {
    "id": 501,
    "first_name": "Jane",
    "last_name": "Doe",
    "display_name": "Jane Doe",
    "email": "jane.doe@example.com",
    "mobile_number": "+1-555-0100",
    "work_number": "+1-555-0101",
    "job_title": "VP of Engineering",
    "sales_accounts": [
      { "id": 101, "name": "Acme Corp", "is_primary": true }
    ],
    "lifecycle_stage_id": 71010794467,
    "created_at": "2026-02-09T10:00:00Z",
    "updated_at": "2026-02-09T10:00:00Z"
  }
}
```

**Get a Contact**

```
GET /api/contacts/{id}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `id` | path | Yes | Contact ID |
| `include` | query | No | Related resources: `sales_account`, `deal`, `tasks`, `appointments`, `notes` |

**Get All Contacts**

```
GET /api/contacts/view/{view_id}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `view_id` | path | Yes | View ID (use predefined or custom views) |
| `page` | query | No | Page number |
| `per_page` | query | No | Records per page (max 100) |
| `sort` | query | No | Sort field |
| `sort_type` | query | No | `asc` or `desc` |

**Update a Contact**

```
PUT /api/contacts/{id}
```

**Request Body:**
```json
{
  "contact": {
    "mobile_number": "+1-555-0200",
    "job_title": "CTO"
  }
}
```

**Delete a Contact**

```
DELETE /api/contacts/{id}
```

**Response (204 No Content)**

#### 6.5.2 Accounts (Sales Accounts)

**Create an Account**

```
POST /api/sales_accounts
```

**Request Body:**
```json
{
  "sales_account": {
    "name": "Acme Corp",
    "website": "https://acme.com",
    "phone": "+1-555-0000",
    "industry_type_id": 71010794470,
    "number_of_employees": 500,
    "annual_revenue": 50000000,
    "address": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "zipcode": "94105",
    "country": "US"
  }
}
```

**Get / Get All / Update / Delete** follow the same pattern at `/api/sales_accounts`.

#### 6.5.3 Deals

**Create a Deal**

```
POST /api/deals
```

**Request Body:**
```json
{
  "deal": {
    "name": "Acme Corp Enterprise License",
    "amount": 150000,
    "sales_account_id": 101,
    "contacts_added_list": [501],
    "deal_stage_id": 71010794471,
    "deal_pipeline_id": 71010794472,
    "expected_close": "2026-06-30",
    "deal_type_id": 71010794473,
    "currency_id": 1
  }
}
```

**Get / Get All / Update / Delete** follow the same pattern at `/api/deals`.

#### 6.5.4 Notes

**Create a Note**

```
POST /api/contacts/{contact_id}/notes
```

**Request Body:**
```json
{
  "note": {
    "description": "Jane confirmed budget allocation for Q2. Proceed with enterprise pricing proposal."
  }
}
```

**Update a Note**

```
PUT /api/contacts/{contact_id}/notes/{note_id}
```

**Delete a Note**

```
DELETE /api/contacts/{contact_id}/notes/{note_id}
```

Notes can also be created against deals (`/api/deals/{deal_id}/notes`) and accounts (`/api/sales_accounts/{account_id}/notes`).

#### 6.5.5 Tasks

**Create a Task**

```
POST /api/tasks
```

**Request Body:**
```json
{
  "task": {
    "title": "Follow up with Jane Doe",
    "description": "Send enterprise pricing document",
    "due_date": "2026-02-15T14:00:00Z",
    "owner_id": 1001,
    "targetable_type": "Contact",
    "targetable_id": 501,
    "task_type_id": 71010794474,
    "status": 0
  }
}
```

**Get a Task**

```
GET /api/tasks/{id}
```

**Get All Tasks**

```
GET /api/tasks?filter=open&page=1&per_page=25
```

**Update a Task**

```
PUT /api/tasks/{id}
```

**Delete a Task**

```
DELETE /api/tasks/{id}
```

#### 6.5.6 Appointments

**Create an Appointment**

```
POST /api/appointments
```

**Request Body:**
```json
{
  "appointment": {
    "title": "Demo call with Acme Corp",
    "description": "Product demo for enterprise features",
    "from_date": "2026-02-12T14:00:00Z",
    "end_date": "2026-02-12T15:00:00Z",
    "time_zone": "America/Los_Angeles",
    "targetable_type": "Contact",
    "targetable_id": 501,
    "location": "Zoom",
    "is_allday": false
  }
}
```

**Get / Get All / Update / Delete** follow the same pattern at `/api/appointments`.

#### 6.5.7 Sales Activities

**Get a Sales Activity**

```
GET /api/sales_activities/{id}
```

**Get All Sales Activities**

```
GET /api/sales_activities?page=1&per_page=25
```

### 6.6 Webhooks / Real-time

Freshworks CRM supports webhooks configured via the admin UI:

1. Go to Settings > Admin Settings > Workflows
2. Create a workflow with a webhook action
3. Configure the target URL, HTTP method, and payload

**Webhook Payload (example):**
```json
{
  "event": "onContactCreate",
  "data": {
    "contact": {
      "id": 501,
      "first_name": "Jane",
      "last_name": "Doe",
      "email": "jane.doe@example.com"
    }
  },
  "timestamp": "2026-02-09T10:00:00Z"
}
```

**Esmer Strategy:** Configure webhooks via the Freshworks admin UI. Route webhook payloads through the Esmer backend to push notifications on mobile.

### 6.7 Error Handling

| HTTP Code | Meaning | Esmer Action |
|---|---|---|
| 400 | Bad request / validation error | Display field-level errors |
| 401 | Invalid API key | Prompt user to re-enter API key |
| 403 | Forbidden (feature not in plan) | Notify user about plan limitations |
| 404 | Record not found | Remove from local cache |
| 422 | Unprocessable entity (validation) | Display validation messages |
| 429 | Rate limit exceeded | Backoff; check `Retry-After` header |
| 500 | Internal server error | Retry with exponential backoff |

**Error Response Format:**
```json
{
  "errors": {
    "code": "invalid_value",
    "message": "Validation failed",
    "fields": {
      "email": ["has already been taken"]
    }
  }
}
```

### 6.8 Mobile-Specific Notes

- **Domain-Specific URLs:** Store the user's subdomain during setup. All API calls are domain-specific.
- **View-Based Listing:** Contact listing requires a `view_id`. Fetch available views via `GET /api/contacts/filters` and cache them.
- **Lightweight Notes:** Notes API is nested under the parent resource. Queue note creation offline and replay when connected.
- **Task Notifications:** Sync open tasks with due dates to local notifications. Use `GET /api/tasks?filter=open&include=targetable` for context.
- **API Key Security:** Store the API key in the platform keychain (iOS Keychain / Android Keystore). Do not store in SharedPreferences or UserDefaults.

---

## 7. Intercom (Intercom API v2)

### 7.1 Service Overview

Esmer uses Intercom for users who manage customer communication, support, and engagement. Delegated tasks include:

- Managing contacts (users and leads) for customer relationship tracking
- Creating and managing companies and associating users to them
- Looking up customer profiles before responding to inquiries
- Listing company users for account-level visibility

### 7.2 Authentication

**Method:** API Key (Access Token)

**Header Format:**
```
Authorization: Bearer {access_token}
Accept: application/json
Content-Type: application/json
```

**Obtaining the Access Token:**
1. Create an Intercom developer account
2. Create an app in the Developer Hub
3. Intercom auto-generates an Access Token for the app
4. Use this Access Token as the API key

**Esmer Recommendation:** Store the access token in the device keychain. Tokens do not expire but can be revoked from the Developer Hub. Intercom also supports OAuth 2.0 for marketplace apps, but the API key approach is simpler for Esmer's use case.

### 7.3 Base URL

```
https://api.intercom.io
```

### 7.4 Rate Limits

| Limit | Value |
|---|---|
| Standard rate limit | 1,000 requests per minute |
| Burst allowance | Short bursts above 1,000/min allowed |
| Search endpoint | 20 requests per 10 seconds |
| Scroll endpoint | 1 concurrent scroll per app |

**Rate Limit Headers:**
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 985
X-RateLimit-Reset: 1707436860
```

**Esmer Strategy:** Intercom's limits are generous. Monitor `X-RateLimit-Remaining` and backoff when below 50. Search endpoint has a stricter limit, so cache search results.

### 7.5 API Endpoints

#### 7.5.1 Users (Contacts - type: user)

In Intercom API v2, users and leads are unified under the "Contacts" resource, distinguished by `role` field (`user` or `lead`).

**Create a User**

```
POST /contacts
```

**Request Body:**
```json
{
  "role": "user",
  "external_id": "usr_12345",
  "email": "jane.doe@example.com",
  "name": "Jane Doe",
  "phone": "+1-555-0100",
  "avatar": "https://example.com/jane.jpg",
  "signed_up_at": 1707350400,
  "custom_attributes": {
    "plan": "enterprise",
    "company_size": 500
  }
}
```

**Response (200 OK):**
```json
{
  "type": "contact",
  "id": "6340a04e72cdcc0ed1ae1abc",
  "workspace_id": "abc123xyz",
  "external_id": "usr_12345",
  "role": "user",
  "email": "jane.doe@example.com",
  "name": "Jane Doe",
  "phone": "+1-555-0100",
  "avatar": {
    "type": "avatar",
    "image_url": "https://example.com/jane.jpg"
  },
  "signed_up_at": 1707350400,
  "created_at": 1707436800,
  "updated_at": 1707436800,
  "custom_attributes": {
    "plan": "enterprise",
    "company_size": 500
  },
  "tags": { "type": "list", "data": [] },
  "notes": { "type": "list", "data": [] },
  "companies": { "type": "list", "data": [] }
}
```

**Get a User**

```
GET /contacts/{id}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `id` | path | Yes | Intercom contact ID |

**Get All Users**

```
POST /contacts/search
```

**Request Body:**
```json
{
  "query": {
    "field": "role",
    "operator": "=",
    "value": "user"
  },
  "pagination": {
    "per_page": 50
  }
}
```

**Response (200 OK):**
```json
{
  "type": "list",
  "data": [
    {
      "type": "contact",
      "id": "6340a04e72cdcc0ed1ae1abc",
      "role": "user",
      "email": "jane@example.com",
      "name": "Jane Doe"
    }
  ],
  "total_count": 150,
  "pages": {
    "type": "pages",
    "page": 1,
    "per_page": 50,
    "total_pages": 3,
    "next": {
      "page": 2,
      "starting_after": "WzE2OTc0.."
    }
  }
}
```

**Alternatively, list all contacts:**

```
GET /contacts
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `per_page` | query | No | Results per page (max 150) |
| `starting_after` | query | No | Cursor for next page |

**Update a User**

```
PUT /contacts/{id}
```

**Request Body:**
```json
{
  "name": "Jane A. Doe",
  "phone": "+1-555-0200",
  "custom_attributes": {
    "plan": "enterprise_plus"
  }
}
```

**Delete a User**

```
DELETE /contacts/{id}
```

**Response (200 OK):**
```json
{
  "id": "6340a04e72cdcc0ed1ae1abc",
  "type": "contact",
  "deleted": true
}
```

#### 7.5.2 Leads (Contacts - type: lead)

**Create a Lead**

```
POST /contacts
```

**Request Body:**
```json
{
  "role": "lead",
  "email": "alex@startup.io",
  "name": "Alex Johnson",
  "phone": "+1-555-0300",
  "custom_attributes": {
    "source": "trade_show",
    "interest_level": "high"
  }
}
```

**Get / Get All / Update / Delete** follow the same patterns as Users, filtering by `role=lead` in search queries.

#### 7.5.3 Companies

**Create a Company**

```
POST /companies
```

**Request Body:**
```json
{
  "company_id": "comp_acme_001",
  "name": "Acme Corp",
  "plan": "enterprise",
  "size": 500,
  "website": "https://acme.com",
  "industry": "Technology",
  "custom_attributes": {
    "annual_revenue": 50000000,
    "region": "US-West"
  }
}
```

**Response (200 OK):**
```json
{
  "type": "company",
  "id": "531ee472cce572a6ec000006",
  "company_id": "comp_acme_001",
  "name": "Acme Corp",
  "plan": { "type": "plan", "id": "1", "name": "enterprise" },
  "size": 500,
  "website": "https://acme.com",
  "industry": "Technology",
  "user_count": 0,
  "monthly_spend": 0,
  "created_at": 1707436800,
  "updated_at": 1707436800,
  "custom_attributes": {
    "annual_revenue": 50000000,
    "region": "US-West"
  }
}
```

**Get a Company**

```
GET /companies/{id}
```

**Get All Companies**

```
GET /companies/list
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `page` | query | No | Page number |
| `per_page` | query | No | Items per page (max 60) |
| `order` | query | No | `asc` or `desc` |

**Update a Company**

```
PUT /companies/{id}
```

**List Company Users**

```
GET /companies/{id}/contacts
```

**Response (200 OK):**
```json
{
  "type": "list",
  "data": [
    {
      "type": "contact",
      "id": "6340a04e72cdcc0ed1ae1abc",
      "role": "user",
      "email": "jane@example.com",
      "name": "Jane Doe"
    }
  ],
  "total_count": 12,
  "pages": {
    "type": "pages",
    "page": 1,
    "per_page": 50,
    "total_pages": 1
  }
}
```

**Attach Contact to Company**

```
POST /contacts/{contact_id}/companies
```

**Request Body:**
```json
{
  "id": "531ee472cce572a6ec000006"
}
```

**Detach Contact from Company**

```
DELETE /contacts/{contact_id}/companies/{company_id}
```

### 7.6 Webhooks / Real-time

Intercom supports webhooks configured via the Developer Hub:

**Supported Topics:**
- `contact.created`, `contact.updated`, `contact.deleted`
- `contact.signed_up`, `contact.tag.created`, `contact.tag.deleted`
- `company.created`
- `conversation.created`, `conversation.updated`
- `conversation.user.created`, `conversation.user.replied`
- `conversation.admin.replied`, `conversation.admin.closed`

**Webhook Payload:**
```json
{
  "type": "notification_event",
  "app_id": "abc123xyz",
  "data": {
    "type": "notification_event_data",
    "item": {
      "type": "contact",
      "id": "6340a04e72cdcc0ed1ae1abc",
      "role": "user",
      "email": "jane@example.com",
      "name": "Jane Doe"
    }
  },
  "topic": "contact.created",
  "delivery_status": "pending",
  "delivery_attempts": 1,
  "first_sent_at": 1707436800,
  "created_at": 1707436800
}
```

**Webhook Verification:** Intercom signs payloads with `X-Hub-Signature` using HMAC-SHA1 and the app's client secret.

### 7.7 Error Handling

| HTTP Code | Error Type | Description | Esmer Action |
|---|---|---|---|
| 400 | `bad_request` | Invalid parameters | Fix request parameters |
| 401 | `unauthorized` | Invalid token | Re-enter API key |
| 403 | `forbidden` | Insufficient permissions | Check app permissions |
| 404 | `not_found` | Resource does not exist | Remove from local cache |
| 409 | `conflict` | Duplicate contact | Merge or update existing |
| 422 | `unprocessable_entity` | Validation error | Display validation messages |
| 429 | `rate_limit_exceeded` | Rate limit hit | Wait for `X-RateLimit-Reset` |
| 500 | `server_error` | Intercom server error | Retry with backoff |
| 503 | `service_unavailable` | Intercom maintenance | Retry after delay |

**Error Response Format:**
```json
{
  "type": "error.list",
  "request_id": "req_abc123",
  "errors": [
    {
      "code": "parameter_invalid",
      "message": "email is not a valid email address"
    }
  ]
}
```

### 7.8 Mobile-Specific Notes

- **Unified Contact Model:** Intercom merged users and leads into a single contacts API. Use the `role` field to distinguish in the Esmer UI.
- **Cursor Pagination:** Use `starting_after` cursor-based pagination for contact lists. Do not use page numbers for large datasets.
- **Search API Limits:** The search endpoint is limited to 20 requests per 10 seconds. Cache search results and implement local filtering for rapid lookups.
- **Custom Attributes:** Intercom supports arbitrary custom attributes. Render these dynamically in the contact detail view.
- **Real-time Chat Integration:** For users with Intercom Messenger, consider deep-linking from the Esmer contact view to the Intercom conversation.

---

## 8. Copper (Copper REST API)

### 8.1 Service Overview

Esmer uses Copper for Google Workspace-centric CRM users. Copper integrates deeply with Gmail and Google Calendar. Delegated tasks include:

- Managing persons (contacts) and companies
- Tracking leads through qualification stages
- Managing opportunities (deals) through pipelines
- Creating and tracking tasks and projects
- Retrieving customer sources for analytics

### 8.2 Authentication

**Method:** API Key + Email

**Header Format:**
```
X-PW-AccessToken: {api_key}
X-PW-Application: developer_api
X-PW-UserEmail: {user_email}
Content-Type: application/json
```

**Requirements:**
- API key: Generated from Copper Settings (requires Professional or Business plan)
- Email: The email address of the API key creator

**Esmer Recommendation:** Store both the API key and email in the device keychain. API keys do not expire. All three headers must be present on every request.

### 8.3 Base URL

```
https://api.copper.com/developer_api/v1
```

### 8.4 Rate Limits

| Limit | Value |
|---|---|
| Requests per 10-minute window | 600 |
| Concurrent connections | 36 |

**Rate Limit Headers:**
```
X-PW-REQUEST-LIMIT: 600
X-PW-REQUEST-REMAINING: 585
X-PW-REQUEST-RESET: 2026-02-09T10:10:00Z
```

**Esmer Strategy:** Copper's rate limits are relatively restrictive. Batch operations where possible. Cache entity lists aggressively with a 10-minute TTL.

### 8.5 API Endpoints

#### 8.5.1 People (Persons)

**Create a Person**

```
POST /people
```

**Request Body:**
```json
{
  "name": "Jane Doe",
  "emails": [
    { "email": "jane.doe@example.com", "category": "work" }
  ],
  "phone_numbers": [
    { "number": "+1-555-0100", "category": "mobile" }
  ],
  "company_id": 101,
  "title": "VP of Engineering",
  "address": {
    "street": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "postal_code": "94105",
    "country": "US"
  },
  "details": "Met at TechCrunch Disrupt",
  "tags": ["vip", "enterprise"],
  "socials": [
    { "url": "https://linkedin.com/in/janedoe", "category": "linkedin" }
  ]
}
```

**Response (200 OK):**
```json
{
  "id": 501,
  "name": "Jane Doe",
  "prefix": null,
  "first_name": "Jane",
  "last_name": "Doe",
  "middle_name": null,
  "suffix": null,
  "emails": [
    { "email": "jane.doe@example.com", "category": "work" }
  ],
  "phone_numbers": [
    { "number": "+1-555-0100", "category": "mobile" }
  ],
  "company_id": 101,
  "company_name": "Acme Corp",
  "title": "VP of Engineering",
  "address": {
    "street": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "postal_code": "94105",
    "country": "US"
  },
  "details": "Met at TechCrunch Disrupt",
  "tags": ["vip", "enterprise"],
  "date_created": 1707436800,
  "date_modified": 1707436800,
  "socials": [
    { "url": "https://linkedin.com/in/janedoe", "category": "linkedin" }
  ]
}
```

**Get a Person**

```
GET /people/{id}
```

**Get All People**

```
POST /people/search
```

**Request Body:**
```json
{
  "page_number": 1,
  "page_size": 25,
  "sort_by": "name",
  "sort_direction": "asc"
}
```

Note: Copper uses POST for list/search operations.

**Response (200 OK):**
```json
[
  {
    "id": 501,
    "name": "Jane Doe",
    "emails": [{ "email": "jane@example.com", "category": "work" }],
    "company_name": "Acme Corp"
  }
]
```

**Update a Person**

```
PUT /people/{id}
```

**Request Body:**
```json
{
  "phone_numbers": [
    { "number": "+1-555-0200", "category": "work" }
  ],
  "title": "CTO"
}
```

**Delete a Person**

```
DELETE /people/{id}
```

**Response (200 OK):**
```json
{
  "id": 501,
  "is_deleted": true
}
```

#### 8.5.2 Companies

**Create a Company**

```
POST /companies
```

**Request Body:**
```json
{
  "name": "Acme Corp",
  "email_domain": "acme.com",
  "phone_numbers": [
    { "number": "+1-555-0000", "category": "work" }
  ],
  "address": {
    "street": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "postal_code": "94105",
    "country": "US"
  },
  "details": "Enterprise technology company",
  "tags": ["enterprise", "technology"],
  "socials": [
    { "url": "https://linkedin.com/company/acme", "category": "linkedin" }
  ]
}
```

**Get / Get All (POST /companies/search) / Update / Delete** follow the same pattern as People.

#### 8.5.3 Leads

**Create a Lead**

```
POST /leads
```

**Request Body:**
```json
{
  "name": "Alex Johnson",
  "email": { "email": "alex@startup.io", "category": "work" },
  "phone_numbers": [
    { "number": "+1-555-0300", "category": "mobile" }
  ],
  "company_name": "StartupIO",
  "title": "Founder",
  "details": "Met at trade show. Interested in enterprise plan.",
  "customer_source_id": 10,
  "status": "New",
  "monetary_value": 50000
}
```

**Get / Get All (POST /leads/search) / Update / Delete** follow the same pattern.

#### 8.5.4 Opportunities

**Create an Opportunity**

```
POST /opportunities
```

**Request Body:**
```json
{
  "name": "Acme Corp Enterprise License",
  "company_id": 101,
  "primary_contact_id": 501,
  "pipeline_id": 1,
  "pipeline_stage_id": 10,
  "monetary_value": 150000,
  "close_date": "02/28/2026",
  "customer_source_id": 10,
  "priority": "High",
  "status": "Open",
  "details": "Enterprise license deal from TechCrunch conference"
}
```

**Get / Get All (POST /opportunities/search) / Update / Delete** follow the same pattern.

#### 8.5.5 Tasks

**Create a Task**

```
POST /tasks
```

**Request Body:**
```json
{
  "name": "Follow up with Jane Doe",
  "related_resource": {
    "id": 501,
    "type": "person"
  },
  "assignee_id": 1001,
  "due_date": 1707955200,
  "reminder_date": 1707868800,
  "priority": "High",
  "status": "Open",
  "details": "Send enterprise pricing document."
}
```

**Get / Get All (POST /tasks/search) / Update / Delete** follow the same pattern.

#### 8.5.6 Projects

**Create a Project**

```
POST /projects
```

**Request Body:**
```json
{
  "name": "Acme Corp Onboarding",
  "related_resource": {
    "id": 101,
    "type": "company"
  },
  "assignee_id": 1001,
  "status": "Open",
  "details": "Onboarding process for Acme Corp enterprise account."
}
```

**Get / Get All (POST /projects/search) / Update / Delete** follow the same pattern.

#### 8.5.7 Customer Sources

**Get All Customer Sources**

```
GET /customer_sources
```

**Response (200 OK):**
```json
[
  { "id": 10, "name": "Trade Show" },
  { "id": 11, "name": "Website" },
  { "id": 12, "name": "Referral" },
  { "id": 13, "name": "Cold Call" }
]
```

#### 8.5.8 Users

**Get All Users**

```
GET /users
```

**Response (200 OK):**
```json
[
  {
    "id": 1001,
    "name": "Sales Rep",
    "email": "sales@company.com"
  }
]
```

### 8.6 Webhooks / Real-time

Copper supports webhooks:

**Create a Webhook**

```
POST /webhooks
```

**Request Body:**
```json
{
  "target": "https://api.esmer.ai/webhooks/copper",
  "type": "person",
  "event": "update",
  "secret": {
    "secret": "shared_secret_value",
    "key": "X-Esmer-Signature"
  }
}
```

**Supported Types:** `lead`, `person`, `company`, `opportunity`, `project`, `task`

**Supported Events:** `new`, `update`, `delete`

**Webhook Payload:**
```json
{
  "type": "person",
  "event": "update",
  "ids": [501],
  "subscription_id": "wh_12345",
  "updated_attributes": {
    "title": ["VP of Engineering", "CTO"]
  }
}
```

### 8.7 Error Handling

| HTTP Code | Meaning | Esmer Action |
|---|---|---|
| 400 | Bad request | Fix request parameters |
| 401 | Invalid API key or email | Re-enter credentials |
| 403 | Insufficient plan (feature locked) | Notify user about plan requirements |
| 404 | Resource not found | Remove from local cache |
| 422 | Validation error | Display field-level errors |
| 429 | Rate limit exceeded | Wait for `X-PW-REQUEST-RESET` timestamp |
| 500 | Server error | Retry with backoff |

**Error Response Format:**
```json
{
  "status": 422,
  "message": "Validation failed: Name has already been taken"
}
```

### 8.8 Mobile-Specific Notes

- **POST-based Lists:** Copper uses `POST` for search/list operations (not GET). This is unusual but allows complex filtering in the request body.
- **Date Formats:** Copper uses Unix timestamps for most dates but `MM/DD/YYYY` strings for `close_date` on opportunities. Normalize all dates in the Esmer data layer.
- **Three Required Headers:** Every request needs `X-PW-AccessToken`, `X-PW-Application`, and `X-PW-UserEmail`. Create an HTTP interceptor to auto-attach these.
- **Google Workspace Integration:** Copper is deeply integrated with Google. For users who also have Google Contacts connected, implement deduplication logic between Copper people and Google contacts.
- **Tags:** Tags are first-class entities in Copper. Use them for Esmer-specific categorization (e.g., tagging contacts managed via Esmer).

---

## 9. Affinity (Affinity API)

### 9.1 Service Overview

Esmer uses Affinity for relationship intelligence and deal flow management. Affinity is popular with venture capital firms, investment banks, and professional services. Delegated tasks include:

- Managing persons and organizations for relationship tracking
- Managing lists and list entries for deal pipelines and tracking
- Viewing relationship strength and interaction history
- Organizing contacts into custom lists

### 9.2 Authentication

**Method:** API Key

**Header Format:**
```
Authorization: Basic {base64(":api_key")}
```

Note: Affinity uses HTTP Basic auth with an empty username and the API key as the password.

**Requirements:**
- Affinity account at Scale, Advanced, or Enterprise tier
- API key generated from Affinity Settings

**Esmer Recommendation:** Store the API key in the device keychain. Encode as Base64 of `:{api_key}` (colon + key) for Basic auth. API keys do not expire but can be regenerated.

### 9.3 Base URL

```
https://api.affinity.co
```

### 9.4 Rate Limits

| Limit | Value |
|---|---|
| Requests per minute | 900 |
| Burst allowance | Up to 30 requests per second |

**Rate Limit Headers:**
```
X-RateLimit-Limit: 900
X-RateLimit-Remaining: 880
X-RateLimit-Reset: 1707436860
```

**Esmer Strategy:** Affinity's limits are generous for typical CRM usage. Cache list and person data locally with a 5-minute TTL.

### 9.5 API Endpoints

#### 9.5.1 Persons

**Create a Person**

```
POST /persons
```

**Request Body:**
```json
{
  "first_name": "Jane",
  "last_name": "Doe",
  "emails": ["jane.doe@example.com"],
  "phone_numbers": ["+1-555-0100"],
  "organization_ids": [101]
}
```

**Response (200 OK):**
```json
{
  "id": 501,
  "type": 0,
  "first_name": "Jane",
  "last_name": "Doe",
  "emails": ["jane.doe@example.com"],
  "phone_numbers": ["+1-555-0100"],
  "primary_email": "jane.doe@example.com",
  "organization_ids": [101],
  "list_entries": [],
  "interaction_dates": {
    "first_email_date": null,
    "last_email_date": null,
    "first_event_date": null,
    "last_event_date": null
  },
  "interactions": {
    "first_email": null,
    "last_email": null,
    "num_emails": 0
  }
}
```

**Get a Person**

```
GET /persons/{person_id}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `person_id` | path | Yes | Person ID |
| `with_interaction_dates` | query | No | Include interaction dates (default false) |
| `with_interaction_persons` | query | No | Include interaction persons (default false) |

**Response (200 OK):**
```json
{
  "id": 501,
  "type": 0,
  "first_name": "Jane",
  "last_name": "Doe",
  "emails": ["jane.doe@example.com"],
  "phone_numbers": ["+1-555-0100"],
  "primary_email": "jane.doe@example.com",
  "organization_ids": [101],
  "list_entries": [
    {
      "id": 1001,
      "list_id": 10,
      "creator_id": 1,
      "entity_id": 501,
      "entity_type": 0,
      "created_at": "2026-02-09T10:00:00Z"
    }
  ]
}
```

**Get All Persons**

```
GET /persons
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `term` | query | No | Search term (name or email) |
| `page_size` | query | No | Results per page (default 100, max 500) |
| `page_token` | query | No | Pagination cursor |
| `with_interaction_dates` | query | No | Include interaction dates |

**Response (200 OK):**
```json
{
  "persons": [
    {
      "id": 501,
      "first_name": "Jane",
      "last_name": "Doe",
      "primary_email": "jane.doe@example.com",
      "organization_ids": [101]
    }
  ],
  "next_page_token": "eyJsYXN0X2lkIjo1MDJ9"
}
```

**Update a Person**

```
PUT /persons/{person_id}
```

**Request Body:**
```json
{
  "first_name": "Jane",
  "last_name": "Doe",
  "phone_numbers": ["+1-555-0100", "+1-555-0200"],
  "organization_ids": [101, 102]
}
```

**Delete a Person**

```
DELETE /persons/{person_id}
```

**Response (200 OK):**
```json
{
  "success": true
}
```

#### 9.5.2 Organizations

**Create an Organization**

```
POST /organizations
```

**Request Body:**
```json
{
  "name": "Acme Corp",
  "domain": "acme.com",
  "domains": ["acme.com", "acme.io"],
  "person_ids": [501, 502]
}
```

**Response (200 OK):**
```json
{
  "id": 101,
  "name": "Acme Corp",
  "domain": "acme.com",
  "domains": ["acme.com", "acme.io"],
  "global": false,
  "person_ids": [501, 502],
  "list_entries": [],
  "interaction_dates": {
    "first_email_date": null,
    "last_email_date": null
  },
  "interactions": {
    "num_emails": 0
  }
}
```

**Get an Organization**

```
GET /organizations/{organization_id}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `organization_id` | path | Yes | Organization ID |
| `with_interaction_dates` | query | No | Include interaction dates |

**Get All Organizations**

```
GET /organizations
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `term` | query | No | Search term |
| `page_size` | query | No | Results per page (max 500) |
| `page_token` | query | No | Pagination cursor |
| `with_interaction_dates` | query | No | Include interaction dates |

**Update an Organization**

```
PUT /organizations/{organization_id}
```

**Request Body:**
```json
{
  "name": "Acme Corporation",
  "domains": ["acme.com", "acme.io", "acmecorp.com"]
}
```

**Delete an Organization**

```
DELETE /organizations/{organization_id}
```

#### 9.5.3 Lists

**Get a List**

```
GET /lists/{list_id}
```

**Response (200 OK):**
```json
{
  "id": 10,
  "type": 0,
  "name": "Active Deals",
  "public": true,
  "owner_id": 1,
  "list_size": 45,
  "fields": [
    {
      "id": 100,
      "name": "Deal Stage",
      "value_type": 7,
      "allows_multiple": false,
      "dropdown_options": [
        { "id": 1, "text": "Prospecting", "rank": 0, "color": 0 },
        { "id": 2, "text": "Negotiation", "rank": 1, "color": 1 },
        { "id": 3, "text": "Closed Won", "rank": 2, "color": 2 }
      ]
    },
    {
      "id": 101,
      "name": "Deal Value",
      "value_type": 3,
      "allows_multiple": false
    }
  ],
  "creator_id": 1,
  "created_at": "2026-01-01T00:00:00Z"
}
```

**Get All Lists**

```
GET /lists
```

**Response (200 OK):**
```json
[
  {
    "id": 10,
    "type": 0,
    "name": "Active Deals",
    "public": true,
    "list_size": 45
  },
  {
    "id": 11,
    "type": 1,
    "name": "LP Contacts",
    "public": false,
    "list_size": 120
  }
]
```

#### 9.5.4 List Entries

**Create a List Entry**

```
POST /lists/{list_id}/list-entries
```

**Request Body:**
```json
{
  "entity_id": 501,
  "entity_type": 0,
  "creator_id": 1
}
```

`entity_type`: `0` = Person, `1` = Organization, `8` = Opportunity

**Response (200 OK):**
```json
{
  "id": 1001,
  "list_id": 10,
  "creator_id": 1,
  "entity_id": 501,
  "entity_type": 0,
  "entity": {
    "id": 501,
    "first_name": "Jane",
    "last_name": "Doe",
    "primary_email": "jane.doe@example.com"
  },
  "created_at": "2026-02-09T10:00:00Z"
}
```

**Get a List Entry**

```
GET /lists/{list_id}/list-entries/{list_entry_id}
```

**Get All List Entries**

```
GET /lists/{list_id}/list-entries
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `page_size` | query | No | Results per page (max 500) |
| `page_token` | query | No | Pagination cursor |

**Response (200 OK):**
```json
{
  "list_entries": [
    {
      "id": 1001,
      "list_id": 10,
      "entity_id": 501,
      "entity_type": 0,
      "entity": {
        "id": 501,
        "first_name": "Jane",
        "last_name": "Doe"
      },
      "created_at": "2026-02-09T10:00:00Z"
    }
  ],
  "next_page_token": "eyJsYXN0X2lkIjoxMDAyfQ=="
}
```

**Delete a List Entry**

```
DELETE /lists/{list_id}/list-entries/{list_entry_id}
```

#### 9.5.5 Field Values

**Get Field Values for a List Entry**

```
GET /field-values?list_entry_id={list_entry_id}
```

**Response (200 OK):**
```json
[
  {
    "id": 5001,
    "field_id": 100,
    "entity_id": 501,
    "list_entry_id": 1001,
    "value": 1,
    "value_type": 7
  },
  {
    "id": 5002,
    "field_id": 101,
    "entity_id": 501,
    "list_entry_id": 1001,
    "value": 150000,
    "value_type": 3
  }
]
```

**Create a Field Value**

```
POST /field-values
```

**Request Body:**
```json
{
  "field_id": 100,
  "entity_id": 501,
  "list_entry_id": 1001,
  "value": 2
}
```

**Update a Field Value**

```
PUT /field-values/{field_value_id}
```

**Delete a Field Value**

```
DELETE /field-values/{field_value_id}
```

### 9.6 Webhooks / Real-time

Affinity supports webhooks via its subscription API:

**Create a Webhook Subscription**

```
POST /webhook
```

**Request Body:**
```json
{
  "webhook_url": "https://api.esmer.ai/webhooks/affinity",
  "subscriptions": [
    { "type": "person.created" },
    { "type": "person.updated" },
    { "type": "person.deleted" },
    { "type": "organization.created" },
    { "type": "organization.updated" },
    { "type": "list_entry.created" },
    { "type": "list_entry.deleted" },
    { "type": "field_value.created" },
    { "type": "field_value.updated" },
    { "type": "field_value.deleted" }
  ]
}
```

**Webhook Payload:**
```json
{
  "type": "person.updated",
  "body": {
    "id": 501,
    "type": 0,
    "first_name": "Jane",
    "last_name": "Doe",
    "primary_email": "jane.doe@example.com"
  },
  "sent_at": "2026-02-09T10:01:00Z"
}
```

### 9.7 Error Handling

| HTTP Code | Meaning | Esmer Action |
|---|---|---|
| 400 | Bad request / validation error | Fix request parameters |
| 401 | Invalid API key | Re-enter API key |
| 403 | Insufficient plan or permissions | Notify user about tier requirements |
| 404 | Resource not found | Remove from local cache |
| 409 | Conflict (duplicate) | Merge or skip |
| 422 | Unprocessable entity | Display validation errors |
| 429 | Rate limit exceeded | Backoff based on `X-RateLimit-Reset` |
| 500 | Server error | Retry with exponential backoff |

**Error Response Format:**
```json
{
  "type": "error",
  "message": "Person with id 999 not found"
}
```

### 9.8 Mobile-Specific Notes

- **Relationship Intelligence:** Affinity's unique value is its relationship strength data derived from email and calendar interactions. Display `interaction_dates` and `interactions` prominently in the Esmer contact detail view.
- **List-Centric Model:** Affinity revolves around lists rather than a fixed pipeline. Fetch lists first, then list entries. Cache list metadata locally.
- **Field Values:** Custom field values are stored separately from list entries. Fetch them in parallel with list entries and merge client-side.
- **Basic Auth Encoding:** On every request, compute `Base64(":" + apiKey)` for the Authorization header. Cache the encoded value to avoid recomputing.
- **Page Token Pagination:** Use `page_token` for efficient pagination. Do not use offset-based approaches.

---

## 10. Microsoft Dynamics CRM (Dynamics 365 Web API)

### 10.1 Service Overview

Esmer uses Microsoft Dynamics 365 for enterprise users in the Microsoft ecosystem. Delegated tasks include:

- Managing accounts (companies) with full CRUD operations
- Managing contacts, leads, and opportunities
- Creating and tracking activities (tasks, appointments, phone calls)
- Querying data via OData-style filters
- Working with custom entities

Note: The n8n node currently supports Account CRUD. Esmer extends this with the full Dynamics 365 Web API for contacts, leads, opportunities, and activities.

### 10.2 Authentication

**Method:** OAuth 2.0 (Microsoft Identity Platform)

**Authorization Endpoint:**
```
https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/authorize
```

**Token Endpoint:**
```
https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token
```

**Required Setup:**
1. Register application in Azure Portal (Microsoft Application Registration Portal)
2. Select "Accounts in any organizational directory" for supported account types
3. Add redirect URI for OAuth callback
4. Generate a client secret under Certificates & secrets
5. Configure Dynamics domain and region

**Required Permissions/Scopes:**
| Scope | Purpose |
|---|---|
| `https://{org}.crm.dynamics.com/user_impersonation` | Access Dynamics 365 as the signed-in user |
| `offline_access` | Obtain refresh tokens |

Replace `{org}` with the organization's Dynamics domain.

**Region-Specific URLs:**
| Region | CRM URL |
|---|---|
| North America | `https://{org}.crm.dynamics.com` |
| South America | `https://{org}.crm2.dynamics.com` |
| Canada | `https://{org}.crm3.dynamics.com` |
| EMEA | `https://{org}.crm4.dynamics.com` |
| APAC | `https://{org}.crm5.dynamics.com` |
| Australia | `https://{org}.crm6.dynamics.com` |
| Japan | `https://{org}.crm7.dynamics.com` |
| India | `https://{org}.crm8.dynamics.com` |
| UK | `https://{org}.crm11.dynamics.com` |
| France | `https://{org}.crm12.dynamics.com` |

**Esmer Recommendation:** Use OAuth 2.0 with PKCE for mobile. Store the organization URL, tenant ID, and client ID. Access tokens expire after 3600 seconds. Refresh tokens expire based on tenant policy (typically 90 days).

### 10.3 Base URL

```
https://{org}.api.crm.dynamics.com/api/data/v9.2
```

Where `{org}` is the Dynamics 365 organization name. The region suffix (e.g., `crm4` for EMEA) is determined during authentication.

### 10.4 Rate Limits

Microsoft Dynamics 365 uses "Service Protection" limits:

| Limit Type | Value |
|---|---|
| Requests per 5-minute sliding window per user | 6,000 |
| Concurrent requests per user | 52 |
| Request execution time per 5-minute window | 20 minutes cumulative |
| Max request payload size | 16 MB |

**Rate Limit Response:**
When limits are exceeded, Dynamics returns HTTP 429 with a `Retry-After` header (in seconds).

**Esmer Strategy:** Implement per-user request throttling. Use `$select` to minimize response payload size. Batch operations using the `$batch` endpoint.

### 10.5 API Endpoints

#### 10.5.1 Accounts

**Create an Account**

```
POST /api/data/v9.2/accounts
```

**Headers:**
```
Authorization: Bearer {access_token}
Content-Type: application/json
OData-MaxVersion: 4.0
OData-Version: 4.0
Prefer: return=representation
```

**Request Body:**
```json
{
  "name": "Acme Corp",
  "telephone1": "+1-555-0000",
  "websiteurl": "https://acme.com",
  "industrycode": 12,
  "numberofemployees": 500,
  "revenue": 50000000,
  "address1_line1": "123 Main St",
  "address1_city": "San Francisco",
  "address1_stateorprovince": "CA",
  "address1_postalcode": "94105",
  "address1_country": "US",
  "description": "Enterprise technology company",
  "accountcategorycode": 1
}
```

**Response (201 Created):**
```json
{
  "@odata.context": "https://org.api.crm.dynamics.com/api/data/v9.2/$metadata#accounts/$entity",
  "@odata.etag": "W/\"12345678\"",
  "accountid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "name": "Acme Corp",
  "telephone1": "+1-555-0000",
  "websiteurl": "https://acme.com",
  "numberofemployees": 500,
  "revenue": 50000000,
  "createdon": "2026-02-09T10:00:00Z",
  "modifiedon": "2026-02-09T10:00:00Z",
  "_ownerid_value": "11111111-2222-3333-4444-555555555555",
  "statecode": 0,
  "statuscode": 1
}
```

**Get an Account**

```
GET /api/data/v9.2/accounts({accountid})
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `accountid` | path (GUID) | Yes | Account GUID |
| `$select` | query | No | Comma-separated field names |
| `$expand` | query | No | Related entities to include |

**Example:**
```
GET /api/data/v9.2/accounts(a1b2c3d4-e5f6-7890-abcd-ef1234567890)?$select=name,telephone1,websiteurl,revenue&$expand=contact_customer_accounts($select=fullname,emailaddress1)
```

**Get All Accounts**

```
GET /api/data/v9.2/accounts
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `$select` | query | No | Fields to return |
| `$filter` | query | No | OData filter expression |
| `$orderby` | query | No | Sort field and direction |
| `$top` | query | No | Max results (default 5000) |
| `$skip` | query | No | Skip N records |
| `$count` | query | No | Include total count |
| `Prefer` | header | No | `odata.maxpagesize=50` for pagination |

**Example with filter:**
```
GET /api/data/v9.2/accounts?$select=name,telephone1,revenue&$filter=revenue gt 1000000 and statecode eq 0&$orderby=name asc&$top=50
```

**Response (200 OK):**
```json
{
  "@odata.context": "https://org.api.crm.dynamics.com/api/data/v9.2/$metadata#accounts(name,telephone1,revenue)",
  "value": [
    {
      "@odata.etag": "W/\"12345678\"",
      "accountid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "name": "Acme Corp",
      "telephone1": "+1-555-0000",
      "revenue": 50000000
    }
  ],
  "@odata.nextLink": "https://org.api.crm.dynamics.com/api/data/v9.2/accounts?$select=name&$skiptoken=..."
}
```

**Update an Account**

```
PATCH /api/data/v9.2/accounts({accountid})
```

**Request Body:**
```json
{
  "telephone1": "+1-555-9999",
  "revenue": 75000000,
  "description": "Updated enterprise technology company"
}
```

**Response (204 No Content):** Empty body on success.

**Delete an Account**

```
DELETE /api/data/v9.2/accounts({accountid})
```

**Response (204 No Content)**

#### 10.5.2 Contacts

**Create a Contact**

```
POST /api/data/v9.2/contacts
```

**Request Body:**
```json
{
  "firstname": "Jane",
  "lastname": "Doe",
  "emailaddress1": "jane.doe@example.com",
  "telephone1": "+1-555-0100",
  "mobilephone": "+1-555-0101",
  "jobtitle": "VP of Engineering",
  "department": "Engineering",
  "parentcustomerid_account@odata.bind": "/accounts(a1b2c3d4-e5f6-7890-abcd-ef1234567890)",
  "address1_line1": "123 Main St",
  "address1_city": "San Francisco",
  "address1_stateorprovince": "CA",
  "address1_postalcode": "94105",
  "address1_country": "US",
  "description": "Met at TechCrunch Disrupt"
}
```

**Response (201 Created):**
```json
{
  "@odata.context": "https://org.api.crm.dynamics.com/api/data/v9.2/$metadata#contacts/$entity",
  "@odata.etag": "W/\"23456789\"",
  "contactid": "b2c3d4e5-f6a7-8901-bcde-f23456789012",
  "firstname": "Jane",
  "lastname": "Doe",
  "fullname": "Jane Doe",
  "emailaddress1": "jane.doe@example.com",
  "telephone1": "+1-555-0100",
  "mobilephone": "+1-555-0101",
  "jobtitle": "VP of Engineering",
  "createdon": "2026-02-09T10:00:00Z",
  "modifiedon": "2026-02-09T10:00:00Z",
  "statecode": 0,
  "statuscode": 1
}
```

**Get a Contact**

```
GET /api/data/v9.2/contacts({contactid})?$select=fullname,emailaddress1,telephone1,jobtitle
```

**Get All Contacts**

```
GET /api/data/v9.2/contacts?$select=fullname,emailaddress1,telephone1,jobtitle&$filter=statecode eq 0&$orderby=fullname asc&$top=50
```

**Update a Contact**

```
PATCH /api/data/v9.2/contacts({contactid})
```

**Request Body:**
```json
{
  "telephone1": "+1-555-0200",
  "jobtitle": "CTO"
}
```

**Delete a Contact**

```
DELETE /api/data/v9.2/contacts({contactid})
```

#### 10.5.3 Leads

**Create a Lead**

```
POST /api/data/v9.2/leads
```

**Request Body:**
```json
{
  "firstname": "Alex",
  "lastname": "Johnson",
  "emailaddress1": "alex@startup.io",
  "telephone1": "+1-555-0300",
  "companyname": "StartupIO",
  "jobtitle": "Founder",
  "leadsourcecode": 8,
  "leadqualitycode": 1,
  "subject": "Enterprise plan inquiry from trade show",
  "description": "Met at industry conference. High interest in enterprise features.",
  "budgetamount": 50000
}
```

**Response (201 Created):**
```json
{
  "leadid": "c3d4e5f6-a7b8-9012-cdef-345678901234",
  "fullname": "Alex Johnson",
  "emailaddress1": "alex@startup.io",
  "companyname": "StartupIO",
  "subject": "Enterprise plan inquiry from trade show",
  "statecode": 0,
  "statuscode": 1,
  "createdon": "2026-02-09T10:00:00Z"
}
```

**Get / Get All / Update / Delete** follow the same OData pattern at `/api/data/v9.2/leads`.

**Qualify a Lead**

```
POST /api/data/v9.2/leads({leadid})/Microsoft.Dynamics.CRM.QualifyLead
```

**Request Body:**
```json
{
  "CreateAccount": true,
  "CreateContact": true,
  "CreateOpportunity": true,
  "Status": 3
}
```

#### 10.5.4 Opportunities

**Create an Opportunity**

```
POST /api/data/v9.2/opportunities
```

**Request Body:**
```json
{
  "name": "Acme Corp Enterprise License",
  "estimatedvalue": 150000,
  "estimatedclosedate": "2026-06-30",
  "parentaccountid@odata.bind": "/accounts(a1b2c3d4-e5f6-7890-abcd-ef1234567890)",
  "parentcontactid@odata.bind": "/contacts(b2c3d4e5-f6a7-8901-bcde-f23456789012)",
  "stepname": "Prospecting",
  "description": "Enterprise license deal from TechCrunch conference",
  "opportunityratingcode": 1
}
```

**Get / Get All / Update / Delete** follow the same OData pattern at `/api/data/v9.2/opportunities`.

**Close an Opportunity as Won**

```
POST /api/data/v9.2/WinOpportunity
```

**Request Body:**
```json
{
  "Status": 3,
  "OpportunityClose": {
    "opportunityid@odata.bind": "/opportunities(d4e5f6a7-b890-1234-defg-456789012345)",
    "subject": "Won - Acme Corp Enterprise License",
    "actualrevenue": 150000,
    "actualend": "2026-06-15"
  }
}
```

#### 10.5.5 Tasks

**Create a Task**

```
POST /api/data/v9.2/tasks
```

**Request Body:**
```json
{
  "subject": "Follow up with Jane Doe",
  "description": "Send enterprise pricing document and schedule demo.",
  "scheduledend": "2026-02-15T14:00:00Z",
  "prioritycode": 2,
  "regardingobjectid_contact@odata.bind": "/contacts(b2c3d4e5-f6a7-8901-bcde-f23456789012)"
}
```

**Get / Get All / Update / Delete** follow the OData pattern at `/api/data/v9.2/tasks`.

#### 10.5.6 Phone Calls

**Create a Phone Call Activity**

```
POST /api/data/v9.2/phonecalls
```

**Request Body:**
```json
{
  "subject": "Sales call with Jane Doe",
  "phonenumber": "+1-555-0100",
  "description": "Discussed enterprise pricing and timeline.",
  "directioncode": true,
  "scheduledend": "2026-02-10T15:00:00Z",
  "regardingobjectid_contact@odata.bind": "/contacts(b2c3d4e5-f6a7-8901-bcde-f23456789012)"
}
```

#### 10.5.7 Appointments

**Create an Appointment**

```
POST /api/data/v9.2/appointments
```

**Request Body:**
```json
{
  "subject": "Demo call with Acme Corp",
  "description": "Product demo for enterprise features.",
  "scheduledstart": "2026-02-12T14:00:00Z",
  "scheduledend": "2026-02-12T15:00:00Z",
  "location": "Zoom",
  "regardingobjectid_account@odata.bind": "/accounts(a1b2c3d4-e5f6-7890-abcd-ef1234567890)"
}
```

#### 10.5.8 Batch Operations

**$batch Request**

```
POST /api/data/v9.2/$batch
Content-Type: multipart/mixed;boundary=batch_boundary
```

**Request Body:**
```
--batch_boundary
Content-Type: multipart/mixed;boundary=changeset_boundary

--changeset_boundary
Content-Type: application/http
Content-Transfer-Encoding: binary
Content-ID: 1

POST /api/data/v9.2/accounts HTTP/1.1
Content-Type: application/json

{"name":"New Corp","telephone1":"+1-555-1111"}

--changeset_boundary
Content-Type: application/http
Content-Transfer-Encoding: binary
Content-ID: 2

POST /api/data/v9.2/contacts HTTP/1.1
Content-Type: application/json

{"firstname":"New","lastname":"Contact","parentcustomerid_account@odata.bind":"$1"}

--changeset_boundary--
--batch_boundary--
```

#### 10.5.9 FetchXML Queries

For complex queries beyond OData filter capabilities:

```
GET /api/data/v9.2/contacts?fetchXml={url_encoded_fetchxml}
```

**Example FetchXML:**
```xml
<fetch mapping="logical" count="50">
  <entity name="contact">
    <attribute name="fullname" />
    <attribute name="emailaddress1" />
    <attribute name="telephone1" />
    <filter type="and">
      <condition attribute="statecode" operator="eq" value="0" />
      <condition attribute="modifiedon" operator="last-x-days" value="7" />
    </filter>
    <order attribute="fullname" />
  </entity>
</fetch>
```

### 10.6 Webhooks / Real-time

Dynamics 365 supports real-time notifications via:

**Webhooks (Service Endpoints):**
- Registered via the Plugin Registration Tool or Web API
- Triggered by entity create, update, delete events
- Sends JSON or XML payloads to HTTPS endpoints

**Register a Webhook:**
```
POST /api/data/v9.2/serviceendpoints
```

**Request Body:**
```json
{
  "name": "Esmer Contact Webhook",
  "url": "https://api.esmer.ai/webhooks/dynamics",
  "contract": 8,
  "authtype": 5,
  "authvalue": "{\"SASKey\":\"shared_access_key\",\"SASKeyName\":\"key_name\"}"
}
```

**Webhook Payload:**
```json
{
  "MessageName": "Update",
  "PrimaryEntityName": "contact",
  "PrimaryEntityId": "b2c3d4e5-f6a7-8901-bcde-f23456789012",
  "BusinessUnitId": "00000000-0000-0000-0000-000000000001",
  "UserId": "11111111-2222-3333-4444-555555555555",
  "InputParameters": [
    {
      "key": "Target",
      "value": {
        "Attributes": [
          { "key": "telephone1", "value": "+1-555-0200" }
        ]
      }
    }
  ],
  "PostEntityImages": [],
  "OperationCreatedOn": "2026-02-09T10:01:00Z"
}
```

**Azure Service Bus Integration:** Dynamics 365 can also publish events to Azure Service Bus topics for more reliable real-time processing.

### 10.7 Error Handling

| HTTP Code | Error Code | Description | Esmer Action |
|---|---|---|---|
| 400 | `-2147204326` | Invalid field value | Fix request data |
| 400 | `0x80040216` | Required field missing | Prompt user for field |
| 401 | | Token expired | Refresh OAuth token |
| 403 | `0x80048306` | Insufficient privileges | Notify admin |
| 404 | `0x80040217` | Entity not found | Remove from local cache |
| 409 | | Concurrent update conflict | Re-fetch and retry |
| 412 | | ETag mismatch (optimistic concurrency) | Re-fetch, merge, retry |
| 429 | | Service protection limit | Wait for `Retry-After` seconds |
| 500 | | Internal server error | Retry with backoff |
| 503 | | Service unavailable | Retry with backoff |

**Error Response Format:**
```json
{
  "error": {
    "code": "0x80040216",
    "message": "A required field is missing: [lastname]",
    "innererror": {
      "message": "A required field is missing: [lastname]",
      "type": "Microsoft.Crm.CrmException",
      "stacktrace": "..."
    }
  }
}
```

### 10.8 Mobile-Specific Notes

- **OData Queries:** Always use `$select` to limit returned fields. Dynamics returns all fields by default, which creates very large payloads.
- **GUIDs:** All entity IDs are GUIDs (36-character UUIDs). Store them as strings. Use lowercase without braces in API calls.
- **Lookup Bindings:** To set lookup fields (foreign keys), use the `@odata.bind` annotation (e.g., `parentcustomerid_account@odata.bind`). This is unique to Dynamics and requires special handling in the request builder.
- **Region Detection:** During authentication setup, determine the user's Dynamics region and store the correct CRM URL. This affects both the token scope and the API base URL.
- **Optimistic Concurrency:** Use the `@odata.etag` value with `If-Match` headers on PATCH/DELETE operations to prevent overwriting concurrent changes.
- **Offline Sync:** Use the `modifiedon` field with OData filters for incremental sync: `$filter=modifiedon gt 2026-02-08T00:00:00Z`.
- **Batch Operations:** Use the `$batch` endpoint to combine multiple operations into a single HTTP request. This is critical for reducing round trips on mobile networks.
- **FetchXML for Complex Queries:** When OData filters are insufficient (e.g., aggregate queries, linked entity joins), use FetchXML. URL-encode the XML and pass it as a query parameter.

---

## Appendix: Cross-Service Implementation Priority for Esmer

| Priority | Service | Rationale |
|---|---|---|
| P0 (Launch) | Google Contacts | Most users have Google accounts; foundational contact source |
| P0 (Launch) | HubSpot | Most popular CRM for SMBs; comprehensive API |
| P1 (Post-Launch) | Salesforce | Enterprise CRM leader; complex but essential |
| P1 (Post-Launch) | Pipedrive | Popular sales-focused CRM; clean API |
| P2 (Growth) | Zoho CRM | Strong international presence; multi-region complexity |
| P2 (Growth) | Microsoft Dynamics | Enterprise Microsoft ecosystem users |
| P2 (Growth) | Intercom | Customer communication platform; user/lead management |
| P3 (Expansion) | Freshworks CRM | Lightweight CRM; simpler API |
| P3 (Expansion) | Copper | Niche Google Workspace CRM |
| P3 (Expansion) | Affinity | Niche relationship intelligence platform |

---

## Appendix: Common Mobile Patterns Across All Services

### Offline Queue Architecture

All CRM integrations share the same offline queue pattern:

1. User performs action (create contact, update deal, log note) while offline
2. Esmer serializes the operation to a local SQLite queue with timestamp and operation type
3. On reconnect, queue processor replays operations in order
4. For updates, compare local `modified_at` with server `modified_at` to detect conflicts
5. Conflicts are presented to the user for manual resolution

### Authentication Token Management

1. Store all tokens (OAuth access, refresh, API keys) in platform-specific secure storage
2. iOS: Keychain Services with `kSecAttrAccessibleAfterFirstUnlock`
3. Android: Android Keystore with `setUserAuthenticationRequired(false)`
4. Implement a centralized token manager that handles refresh across all services
5. Proactively refresh OAuth tokens at 80% of their TTL to avoid request failures

### Data Caching Strategy

| Data Type | Cache TTL | Storage |
|---|---|---|
| Contact lists | 5 minutes | SQLite |
| Contact details | 10 minutes | SQLite |
| Company/Account lists | 15 minutes | SQLite |
| Deal/Opportunity pipelines | 5 minutes | SQLite |
| Schema/metadata | 24 hours | SQLite |
| Contact photos | 24 hours | Disk cache |
| Search results | 2 minutes | In-memory |

---

*End of Esmer (ESMO) API Integration Specification: Contacts & People (Category 06)*
