# Esmer (ESMO) API Integration Specification: Notes, Documents & Knowledge

> **Category:** 04 - Notes, Documents & Knowledge
> **Version:** 1.0.0
> **Last Updated:** 2026-02-09
> **Status:** Draft

---

## Table of Contents

- [Cross-Service Comparison Matrix](#cross-service-comparison-matrix)
- [1. Google Docs (Google Docs API v1)](#1-google-docs-google-docs-api-v1)
- [2. Google Sheets (Google Sheets API v4)](#2-google-sheets-google-sheets-api-v4)
- [3. Microsoft Excel 365 (Microsoft Graph API)](#3-microsoft-excel-365-microsoft-graph-api)
- [4. Google Slides (Google Slides API v1)](#4-google-slides-google-slides-api-v1)
- [5. Airtable (Airtable REST API)](#5-airtable-airtable-rest-api)

---

## Cross-Service Comparison Matrix

| Feature | Google Docs | Google Sheets | MS Excel 365 | Google Slides | Airtable |
|---|---|---|---|---|---|
| **Auth Method** | OAuth 2.0 / Service Account | OAuth 2.0 / Service Account | OAuth 2.0 | OAuth 2.0 / Service Account | PAT / OAuth 2.0 |
| **Base URL** | `docs.googleapis.com/v1` | `sheets.googleapis.com/v4` | `graph.microsoft.com/v1.0` | `slides.googleapis.com/v1` | `api.airtable.com/v0` |
| **Rate Limit** | 300 req/min/project | 300 req/min/project | 10,000 req/10min | 300 req/min/project | 5 req/sec/base |
| **Webhooks** | Via Google Drive push notifications | Via Google Drive push notifications | Via Microsoft Graph subscriptions | Via Google Drive push notifications | Via webhooks API |
| **Real-time** | No native (use Drive watch) | No native (use Drive watch) | Subscriptions with change notifications | No native (use Drive watch) | Webhook-based polling |
| **Batch Operations** | Yes (batchUpdate) | Yes (batchUpdate, batchGet) | Yes (batch JSON request) | Yes (batchUpdate) | Yes (up to 10 records) |
| **Offline Support** | Read cached; queue writes | Read cached; queue writes | Read cached; queue writes | Read cached; queue writes | Read cached; queue writes |
| **Max Payload** | 10 MB document | 10 million cells | 5 MB workbook via API | 100 MB presentation | 100,000 records/base |
| **Mobile Priority** | High - note creation | Critical - data entry | High - data entry | Medium - presentation view | High - structured data |

---

## 1. Google Docs (Google Docs API v1)

### 1.1 Service Overview

Esmer uses Google Docs for delegated document operations on behalf of users: creating new documents from templates or scratch, reading document content for summarization, performing find-and-replace across documents, inserting structured text (tables, lists, headings), and applying batch edits that combine multiple changes in a single request. Mobile use cases include quick note capture, voice-to-document transcription, and automated report generation.

### 1.2 Authentication

**Method:** OAuth 2.0 (recommended) or Google Service Account

**OAuth 2.0 Flow:** Authorization Code with PKCE (mobile-appropriate)

**Token Endpoint:** `https://oauth2.googleapis.com/token`
**Authorization Endpoint:** `https://accounts.google.com/o/oauth2/v2/auth`

| Scope | Description | Required For |
|---|---|---|
| `https://www.googleapis.com/auth/documents` | Full read/write access to Google Docs | Create, update, delete documents |
| `https://www.googleapis.com/auth/documents.readonly` | Read-only access to Google Docs | Get document content |
| `https://www.googleapis.com/auth/drive` | Full access to Google Drive (needed for delete) | Delete documents, manage permissions |
| `https://www.googleapis.com/auth/drive.file` | Access to files created/opened by the app | Minimum scope for file operations |

**Recommendation for Esmer:** Use `drive.file` + `documents` scopes. The `drive.file` scope limits access to only files the app creates or the user explicitly opens through the app, which aligns with the principle of least privilege for a mobile delegation assistant.

**Token Refresh:** Access tokens expire after 3600 seconds. Esmer must implement automatic refresh using the refresh token. Store refresh tokens securely in the device keychain (iOS Keychain / Android Keystore).

### 1.3 Base URL

```
https://docs.googleapis.com/v1
```

### 1.4 Rate Limits

| Quota | Limit | Window |
|---|---|---|
| Read requests | 300 per minute per project | Per minute |
| Write requests | 60 per minute per user per project | Per minute |
| Per-user rate limit | 60 requests per minute | Per minute |
| Daily limit | Varies by project quota (default ~unlimited with billing) | Per day |

**Esmer Strategy:** Implement exponential backoff starting at 1 second. Batch multiple edits into single `batchUpdate` calls. Cache document metadata locally for 5 minutes.

### 1.5 API Endpoints

#### 1.5.1 Documents Resource

##### Create a Document

```
POST /v1/documents
```

**Description:** Creates a new blank Google Docs document with the specified title.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `title` | body | string | Yes | The title of the document |

**Request:**

```json
{
  "title": "Esmer Meeting Notes - 2026-02-09"
}
```

**Response (200 OK):**

```json
{
  "documentId": "1BxiMVs0XRA5nFMdKvBdBZjgmUii3vXg7lLKf2tT8Eo0",
  "title": "Esmer Meeting Notes - 2026-02-09",
  "body": {
    "content": [
      {
        "endIndex": 1,
        "sectionBreak": {
          "sectionStyle": {
            "columnSeparatorStyle": "NONE",
            "contentDirection": "LEFT_TO_RIGHT"
          }
        }
      },
      {
        "startIndex": 1,
        "endIndex": 2,
        "paragraph": {
          "elements": [
            {
              "startIndex": 1,
              "endIndex": 2,
              "textRun": {
                "content": "\n",
                "textStyle": {}
              }
            }
          ],
          "paragraphStyle": {
            "namedStyleType": "NORMAL_TEXT",
            "direction": "LEFT_TO_RIGHT"
          }
        }
      }
    ]
  },
  "revisionId": "AOV_f4...",
  "documentStyle": {},
  "namedStyles": {},
  "suggestionsViewMode": "SUGGESTIONS_INLINE"
}
```

**Error Codes:**

| Code | Description |
|---|---|
| 400 | Invalid request (e.g., missing title) |
| 401 | Invalid or expired OAuth token |
| 403 | Insufficient scopes or quota exceeded |
| 429 | Rate limit exceeded |

##### Get a Document

```
GET /v1/documents/{documentId}
```

**Description:** Retrieves the full content of a document including body, headers, footers, and structural elements.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `documentId` | path | string | Yes | The ID of the document to retrieve |
| `suggestionsViewMode` | query | enum | No | Mode for viewing suggestions: `DEFAULT_FOR_CURRENT_ACCESS`, `SUGGESTIONS_INLINE`, `PREVIEW_SUGGESTIONS_ACCEPTED`, `PREVIEW_WITHOUT_SUGGESTIONS` |

**Request:**

```
GET /v1/documents/1BxiMVs0XRA5nFMdKvBdBZjgmUii3vXg7lLKf2tT8Eo0
```

**Response (200 OK):**

```json
{
  "documentId": "1BxiMVs0XRA5nFMdKvBdBZjgmUii3vXg7lLKf2tT8Eo0",
  "title": "Esmer Meeting Notes - 2026-02-09",
  "revisionId": "AOV_f4...",
  "body": {
    "content": [
      {
        "endIndex": 1,
        "sectionBreak": {
          "sectionStyle": {
            "columnSeparatorStyle": "NONE",
            "contentDirection": "LEFT_TO_RIGHT"
          }
        }
      },
      {
        "startIndex": 1,
        "endIndex": 25,
        "paragraph": {
          "elements": [
            {
              "startIndex": 1,
              "endIndex": 25,
              "textRun": {
                "content": "Meeting agenda items:\n",
                "textStyle": {
                  "bold": false,
                  "fontSize": {
                    "magnitude": 11,
                    "unit": "PT"
                  }
                }
              }
            }
          ],
          "paragraphStyle": {
            "namedStyleType": "NORMAL_TEXT"
          }
        }
      }
    ]
  },
  "documentStyle": {
    "pageSize": {
      "height": { "magnitude": 792, "unit": "PT" },
      "width": { "magnitude": 612, "unit": "PT" }
    },
    "marginTop": { "magnitude": 72, "unit": "PT" },
    "marginBottom": { "magnitude": 72, "unit": "PT" },
    "marginLeft": { "magnitude": 72, "unit": "PT" },
    "marginRight": { "magnitude": 72, "unit": "PT" }
  }
}
```

**Error Codes:**

| Code | Description |
|---|---|
| 404 | Document not found or no access |
| 401 | Invalid or expired OAuth token |
| 403 | Insufficient permissions |

##### Batch Update a Document

```
POST /v1/documents/{documentId}:batchUpdate
```

**Description:** Applies one or more structured edits to a document atomically. This is the primary write endpoint -- all document mutations use this method.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `documentId` | path | string | Yes | The document ID |
| `requests` | body | array | Yes | Array of update request objects |
| `writeControl` | body | object | No | Revision-based write control |

**Available Request Types:**

| Request Type | Description |
|---|---|
| `insertText` | Insert text at a specified location |
| `deleteContentRange` | Delete content within a range |
| `replaceAllText` | Find and replace text throughout the document |
| `insertTable` | Insert a table at a location |
| `insertTableRow` | Insert a row in a table |
| `insertTableColumn` | Insert a column in a table |
| `deleteTableRow` | Delete a row from a table |
| `deleteTableColumn` | Delete a column from a table |
| `insertInlineImage` | Insert an image inline |
| `insertPageBreak` | Insert a page break |
| `createParagraphBullets` | Add bullet/numbered list formatting |
| `deleteParagraphBullets` | Remove bullet formatting |
| `updateTextStyle` | Update bold, italic, font, color, etc. |
| `updateParagraphStyle` | Update alignment, spacing, heading level |
| `createNamedRange` | Create a named range for bookmarking |
| `deleteNamedRange` | Delete a named range |
| `updateDocumentStyle` | Update page margins, size, etc. |
| `createHeader` | Create a header section |
| `createFooter` | Create a footer section |
| `deleteHeader` | Delete a header |
| `deleteFooter` | Delete a footer |
| `mergeTableCells` | Merge cells in a table |
| `unmergeTableCells` | Unmerge previously merged cells |
| `updateTableColumnProperties` | Update column width |
| `updateTableRowStyle` | Update row height |
| `replaceImage` | Replace an existing image |
| `updateSectionStyle` | Update section-level styling |
| `insertSectionBreak` | Insert a section break |
| `deleteSectionBreak` (implied via deleteContentRange) | Remove section break |

**Request Example -- Insert Text:**

```json
{
  "requests": [
    {
      "insertText": {
        "text": "Action Items:\n1. Review budget proposal\n2. Schedule follow-up meeting\n",
        "location": {
          "index": 1
        }
      }
    }
  ]
}
```

**Request Example -- Replace All Text:**

```json
{
  "requests": [
    {
      "replaceAllText": {
        "containsText": {
          "text": "{{MEETING_DATE}}",
          "matchCase": true
        },
        "replaceText": "February 9, 2026"
      }
    }
  ]
}
```

**Request Example -- Insert Table:**

```json
{
  "requests": [
    {
      "insertTable": {
        "rows": 3,
        "columns": 4,
        "location": {
          "index": 1
        }
      }
    }
  ]
}
```

**Request Example -- Multiple Operations (Batch):**

```json
{
  "requests": [
    {
      "insertText": {
        "text": "Weekly Status Report\n",
        "location": { "index": 1 }
      }
    },
    {
      "updateParagraphStyle": {
        "range": {
          "startIndex": 1,
          "endIndex": 22
        },
        "paragraphStyle": {
          "namedStyleType": "HEADING_1",
          "alignment": "CENTER"
        },
        "fields": "namedStyleType,alignment"
      }
    },
    {
      "insertText": {
        "text": "Prepared by Esmer AI Assistant\n\n",
        "location": { "index": 22 }
      }
    },
    {
      "updateTextStyle": {
        "range": {
          "startIndex": 22,
          "endIndex": 52
        },
        "textStyle": {
          "italic": true,
          "fontSize": {
            "magnitude": 10,
            "unit": "PT"
          },
          "foregroundColor": {
            "color": {
              "rgbColor": {
                "red": 0.5,
                "green": 0.5,
                "blue": 0.5
              }
            }
          }
        },
        "fields": "italic,fontSize,foregroundColor"
      }
    },
    {
      "insertTable": {
        "rows": 4,
        "columns": 3,
        "location": { "index": 54 }
      }
    }
  ],
  "writeControl": {
    "requiredRevisionId": "AOV_f4..."
  }
}
```

**Response (200 OK):**

```json
{
  "documentId": "1BxiMVs0XRA5nFMdKvBdBZjgmUii3vXg7lLKf2tT8Eo0",
  "replies": [
    {},
    {},
    {},
    {},
    {
      "insertInlineImage": {
        "objectId": "kix.abc123"
      }
    }
  ],
  "writeControl": {
    "requiredRevisionId": "AOV_f5..."
  }
}
```

**Request Example -- Delete Content Range:**

```json
{
  "requests": [
    {
      "deleteContentRange": {
        "range": {
          "startIndex": 10,
          "endIndex": 50
        }
      }
    }
  ]
}
```

**Request Example -- Create Bullet List:**

```json
{
  "requests": [
    {
      "createParagraphBullets": {
        "range": {
          "startIndex": 1,
          "endIndex": 100
        },
        "bulletPreset": "BULLET_DISC_CIRCLE_SQUARE"
      }
    }
  ]
}
```

**Error Codes:**

| Code | Description |
|---|---|
| 400 | Invalid request (e.g., index out of bounds, malformed request) |
| 401 | Authentication failure |
| 403 | Insufficient permissions or quota exceeded |
| 404 | Document not found |
| 409 | Conflict -- revision mismatch when using `writeControl` |
| 429 | Rate limit exceeded |

### 1.6 Webhooks / Real-time

Google Docs does not provide native webhook or push notification support. To detect document changes, use the **Google Drive API** push notifications channel:

```
POST https://www.googleapis.com/drive/v3/files/{fileId}/watch
```

```json
{
  "id": "esmer-docs-channel-unique-id",
  "type": "web_hook",
  "address": "https://api.esmer.app/webhooks/google-drive",
  "expiration": 1739145600000
}
```

**Notification Payload (headers):**

| Header | Description |
|---|---|
| `X-Goog-Channel-ID` | The channel ID you set |
| `X-Goog-Resource-ID` | Opaque ID for the watched resource |
| `X-Goog-Resource-State` | `sync`, `update`, `trash`, etc. |
| `X-Goog-Changed` | Comma-separated: `content`, `properties`, `permissions` |

Channels expire (max 24 hours for most resources). Esmer must renew channels before expiration.

### 1.7 Error Handling

| HTTP Code | Meaning | Esmer Action |
|---|---|---|
| 400 | Bad request | Log error, fix request, do not retry |
| 401 | Token expired | Refresh token and retry once |
| 403 | Permission denied or quota | Check scopes; if quota, backoff and retry |
| 404 | Document not found | Notify user, remove from local cache |
| 409 | Revision conflict | Re-fetch document, rebase edits, retry |
| 429 | Rate limited | Exponential backoff: 1s, 2s, 4s, 8s, max 60s |
| 500 | Server error | Retry with exponential backoff up to 3 times |
| 503 | Service unavailable | Retry with exponential backoff up to 5 times |

### 1.8 Mobile-Specific Notes

- **Index Tracking:** Document edits use character indices. When performing multiple sequential edits, compute indices in reverse order (end-to-start) to avoid index shifting, or batch all edits into a single `batchUpdate` call.
- **Offline Queue:** Queue `batchUpdate` requests when offline. On reconnect, fetch latest `revisionId`, rebase indices if the document changed, and apply the queued edits.
- **Payload Size:** Full document GET responses can be large. For display purposes, consider requesting only specific fields using the `fields` query parameter: `?fields=documentId,title,body.content`.
- **Voice Input:** Map voice-transcribed text directly to `insertText` requests. Use `endOfSegmentLocation` to append at the end of the document body.
- **Template Workflow:** Pre-create template documents with placeholder tokens like `{{NAME}}`, `{{DATE}}`. Use `replaceAllText` to fill them in one batch call.

---

## 2. Google Sheets (Google Sheets API v4)

### 2.1 Service Overview

Esmer uses Google Sheets as the primary structured-data interface for mobile users: reading spreadsheet data for dashboards, appending rows for data entry (expense tracking, time logging, CRM entries), updating existing records by matching column values, clearing data ranges, and managing sheets within spreadsheets. Key mobile workflows include quick data capture via voice/form, automated report population, and reading tabular data for AI analysis.

### 2.2 Authentication

**Method:** OAuth 2.0 (recommended) or Google Service Account

**OAuth 2.0 Flow:** Authorization Code with PKCE

**Token Endpoint:** `https://oauth2.googleapis.com/token`
**Authorization Endpoint:** `https://accounts.google.com/o/oauth2/v2/auth`

| Scope | Description | Required For |
|---|---|---|
| `https://www.googleapis.com/auth/spreadsheets` | Full read/write access to Google Sheets | All write operations |
| `https://www.googleapis.com/auth/spreadsheets.readonly` | Read-only access | Read-only scenarios |
| `https://www.googleapis.com/auth/drive` | Full Drive access | Delete spreadsheets |
| `https://www.googleapis.com/auth/drive.file` | Access only app-created/opened files | Minimum scope for file operations |

**Recommendation for Esmer:** Use `spreadsheets` + `drive.file` for standard operations. Only request `drive` scope when the user explicitly needs to delete spreadsheets.

### 2.3 Base URL

```
https://sheets.googleapis.com/v4
```

### 2.4 Rate Limits

| Quota | Limit | Window |
|---|---|---|
| Read requests | 300 per minute per project | Per minute |
| Write requests | 300 per minute per project | Per minute |
| Per-user rate limit | 60 requests per minute per user | Per minute |
| Max cells per spreadsheet | 10,000,000 | N/A |

**Esmer Strategy:** Use batch reads/writes to consolidate API calls. Cache sheet metadata and header rows locally. Implement per-user request throttling on the client side.

### 2.5 API Endpoints

#### 2.5.1 Spreadsheets Resource

##### Create a Spreadsheet

```
POST /v4/spreadsheets
```

**Description:** Creates a new spreadsheet with optional sheets and locale settings.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `properties.title` | body | string | Yes | Title of the spreadsheet |
| `properties.locale` | body | string | No | Locale (e.g., `en_US`) |
| `properties.autoRecalc` | body | enum | No | `ON_CHANGE`, `MINUTE`, `HOUR` |
| `sheets` | body | array | No | Array of sheet objects to create |

**Request:**

```json
{
  "properties": {
    "title": "Esmer Expense Tracker - Feb 2026",
    "locale": "en_US",
    "autoRecalc": "ON_CHANGE"
  },
  "sheets": [
    {
      "properties": {
        "title": "Expenses",
        "index": 0
      }
    },
    {
      "properties": {
        "title": "Summary",
        "index": 1
      }
    }
  ]
}
```

**Response (200 OK):**

```json
{
  "spreadsheetId": "1BxiMVs0XRA5nFMdKvBdBZjgmUii3vXg7lLKf2tT8Eo0",
  "properties": {
    "title": "Esmer Expense Tracker - Feb 2026",
    "locale": "en_US",
    "autoRecalc": "ON_CHANGE",
    "timeZone": "America/New_York",
    "defaultFormat": {}
  },
  "sheets": [
    {
      "properties": {
        "sheetId": 0,
        "title": "Expenses",
        "index": 0,
        "sheetType": "GRID",
        "gridProperties": {
          "rowCount": 1000,
          "columnCount": 26
        }
      }
    },
    {
      "properties": {
        "sheetId": 1,
        "title": "Summary",
        "index": 1,
        "sheetType": "GRID",
        "gridProperties": {
          "rowCount": 1000,
          "columnCount": 26
        }
      }
    }
  ],
  "spreadsheetUrl": "https://docs.google.com/spreadsheets/d/1BxiMVs0XRA5nFMdKvBdBZjgmUii3vXg7lLKf2tT8Eo0/edit"
}
```

##### Get a Spreadsheet

```
GET /v4/spreadsheets/{spreadsheetId}
```

**Description:** Returns the spreadsheet metadata including sheet properties, named ranges, and optionally cell data.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `spreadsheetId` | path | string | Yes | The spreadsheet ID |
| `ranges` | query | string[] | No | Ranges to include (A1 notation) |
| `includeGridData` | query | boolean | No | Include cell data in response |
| `fields` | query | string | No | Field mask for partial response |

**Request:**

```
GET /v4/spreadsheets/1BxiMVs0XRA5nFMdKvBdBZjgmUii3vXg7lLKf2tT8Eo0?fields=spreadsheetId,properties.title,sheets.properties
```

**Response (200 OK):**

```json
{
  "spreadsheetId": "1BxiMVs0XRA5nFMdKvBdBZjgmUii3vXg7lLKf2tT8Eo0",
  "properties": {
    "title": "Esmer Expense Tracker - Feb 2026"
  },
  "sheets": [
    {
      "properties": {
        "sheetId": 0,
        "title": "Expenses",
        "index": 0,
        "sheetType": "GRID",
        "gridProperties": {
          "rowCount": 1000,
          "columnCount": 26
        }
      }
    }
  ]
}
```

##### Delete a Spreadsheet

```
DELETE https://www.googleapis.com/drive/v3/files/{fileId}
```

**Description:** Deletes a spreadsheet permanently. This uses the Google Drive API since Sheets API does not have a native delete endpoint.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `fileId` | path | string | Yes | The spreadsheet ID (same as fileId) |

**Response:** `204 No Content`

#### 2.5.2 Values Resource (Cell Read/Write)

##### Read Cell Values (Single Range)

```
GET /v4/spreadsheets/{spreadsheetId}/values/{range}
```

**Description:** Returns cell values for a specified range in A1 notation.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `spreadsheetId` | path | string | Yes | The spreadsheet ID |
| `range` | path | string | Yes | Range in A1 notation (e.g., `Sheet1!A1:D10`) |
| `majorDimension` | query | enum | No | `ROWS` (default) or `COLUMNS` |
| `valueRenderOption` | query | enum | No | `FORMATTED_VALUE`, `UNFORMATTED_VALUE`, `FORMULA` |
| `dateTimeRenderOption` | query | enum | No | `SERIAL_NUMBER`, `FORMATTED_STRING` |

**Request:**

```
GET /v4/spreadsheets/1BxiMVs0XRA.../values/Expenses!A1:E5?valueRenderOption=FORMATTED_VALUE
```

**Response (200 OK):**

```json
{
  "range": "Expenses!A1:E5",
  "majorDimension": "ROWS",
  "values": [
    ["Date", "Category", "Description", "Amount", "Receipt"],
    ["2026-02-01", "Food", "Team lunch", "$45.50", "Yes"],
    ["2026-02-03", "Transport", "Uber to office", "$12.00", "Yes"],
    ["2026-02-05", "Software", "Figma subscription", "$15.00", "No"],
    ["2026-02-07", "Food", "Client dinner", "$120.00", "Yes"]
  ]
}
```

##### Read Cell Values (Multiple Ranges -- Batch Get)

```
GET /v4/spreadsheets/{spreadsheetId}/values:batchGet
```

**Description:** Returns cell values for multiple ranges in a single request.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `spreadsheetId` | path | string | Yes | The spreadsheet ID |
| `ranges` | query | string[] | Yes | Multiple ranges in A1 notation |
| `majorDimension` | query | enum | No | `ROWS` or `COLUMNS` |
| `valueRenderOption` | query | enum | No | `FORMATTED_VALUE`, `UNFORMATTED_VALUE`, `FORMULA` |

**Request:**

```
GET /v4/spreadsheets/1BxiMVs0XRA.../values:batchGet?ranges=Expenses!A1:E1&ranges=Summary!A1:B5
```

**Response (200 OK):**

```json
{
  "spreadsheetId": "1BxiMVs0XRA...",
  "valueRanges": [
    {
      "range": "Expenses!A1:E1",
      "majorDimension": "ROWS",
      "values": [["Date", "Category", "Description", "Amount", "Receipt"]]
    },
    {
      "range": "Summary!A1:B5",
      "majorDimension": "ROWS",
      "values": [
        ["Category", "Total"],
        ["Food", "$165.50"],
        ["Transport", "$12.00"],
        ["Software", "$15.00"]
      ]
    }
  ]
}
```

##### Write Cell Values (Single Range)

```
PUT /v4/spreadsheets/{spreadsheetId}/values/{range}
```

**Description:** Writes values to a specified range, replacing existing data.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `spreadsheetId` | path | string | Yes | The spreadsheet ID |
| `range` | path | string | Yes | Range in A1 notation |
| `valueInputOption` | query | enum | Yes | `RAW` (as-is) or `USER_ENTERED` (parsed like UI input) |
| `includeValuesInResponse` | query | boolean | No | Return updated values |
| `body.values` | body | array[] | Yes | 2D array of values |

**Request:**

```
PUT /v4/spreadsheets/1BxiMVs0XRA.../values/Expenses!A6:E6?valueInputOption=USER_ENTERED
```

```json
{
  "values": [
    ["2026-02-09", "Office", "Printer paper", "$25.00", "No"]
  ]
}
```

**Response (200 OK):**

```json
{
  "spreadsheetId": "1BxiMVs0XRA...",
  "updatedRange": "Expenses!A6:E6",
  "updatedRows": 1,
  "updatedColumns": 5,
  "updatedCells": 5
}
```

##### Write Cell Values (Batch Update)

```
POST /v4/spreadsheets/{spreadsheetId}/values:batchUpdate
```

**Description:** Updates multiple ranges of cells in a single request.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `spreadsheetId` | path | string | Yes | The spreadsheet ID |
| `valueInputOption` | body | enum | Yes | `RAW` or `USER_ENTERED` |
| `data` | body | array | Yes | Array of ValueRange objects |

**Request:**

```json
{
  "valueInputOption": "USER_ENTERED",
  "data": [
    {
      "range": "Expenses!A6:E6",
      "values": [["2026-02-09", "Office", "Printer paper", "$25.00", "No"]]
    },
    {
      "range": "Summary!B5",
      "values": [["$25.00"]]
    }
  ]
}
```

**Response (200 OK):**

```json
{
  "spreadsheetId": "1BxiMVs0XRA...",
  "totalUpdatedRows": 2,
  "totalUpdatedColumns": 6,
  "totalUpdatedCells": 6,
  "totalUpdatedSheets": 2,
  "responses": [
    {
      "spreadsheetId": "1BxiMVs0XRA...",
      "updatedRange": "Expenses!A6:E6",
      "updatedRows": 1,
      "updatedColumns": 5,
      "updatedCells": 5
    },
    {
      "spreadsheetId": "1BxiMVs0XRA...",
      "updatedRange": "Summary!B5",
      "updatedRows": 1,
      "updatedColumns": 1,
      "updatedCells": 1
    }
  ]
}
```

##### Append Values

```
POST /v4/spreadsheets/{spreadsheetId}/values/{range}:append
```

**Description:** Appends values after the last row of data in a range. This is the primary data-entry endpoint for Esmer mobile input.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `spreadsheetId` | path | string | Yes | The spreadsheet ID |
| `range` | path | string | Yes | Range defining the table to append to (e.g., `Sheet1!A:E`) |
| `valueInputOption` | query | enum | Yes | `RAW` or `USER_ENTERED` |
| `insertDataOption` | query | enum | No | `OVERWRITE` (default) or `INSERT_ROWS` |
| `includeValuesInResponse` | query | boolean | No | Return appended values |
| `body.values` | body | array[] | Yes | 2D array of values to append |

**Request:**

```
POST /v4/spreadsheets/1BxiMVs0XRA.../values/Expenses!A:E:append?valueInputOption=USER_ENTERED&insertDataOption=INSERT_ROWS
```

```json
{
  "values": [
    ["2026-02-09", "Food", "Coffee meeting", "$8.50", "No"],
    ["2026-02-09", "Transport", "Parking", "$5.00", "Yes"]
  ]
}
```

**Response (200 OK):**

```json
{
  "spreadsheetId": "1BxiMVs0XRA...",
  "tableRange": "Expenses!A1:E6",
  "updates": {
    "spreadsheetId": "1BxiMVs0XRA...",
    "updatedRange": "Expenses!A7:E8",
    "updatedRows": 2,
    "updatedColumns": 5,
    "updatedCells": 10
  }
}
```

##### Clear Values

```
POST /v4/spreadsheets/{spreadsheetId}/values/{range}:clear
```

**Description:** Clears values from a specified range without deleting the cells themselves.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `spreadsheetId` | path | string | Yes | The spreadsheet ID |
| `range` | path | string | Yes | Range to clear in A1 notation |

**Request:**

```
POST /v4/spreadsheets/1BxiMVs0XRA.../values/Expenses!A2:E100:clear
```

```json
{}
```

**Response (200 OK):**

```json
{
  "spreadsheetId": "1BxiMVs0XRA...",
  "clearedRange": "Expenses!A2:E8"
}
```

#### 2.5.3 Sheets Resource (Sheet Management)

##### Batch Update (Sheet Operations)

```
POST /v4/spreadsheets/{spreadsheetId}:batchUpdate
```

**Description:** Applies structural changes to a spreadsheet: adding/deleting sheets, deleting rows/columns, formatting, etc. This is distinct from `values:batchUpdate` which handles cell values.

**Add a Sheet Request:**

```json
{
  "requests": [
    {
      "addSheet": {
        "properties": {
          "title": "March Expenses",
          "index": 2,
          "tabColorStyle": {
            "rgbColor": {
              "red": 0.2,
              "green": 0.7,
              "blue": 0.3
            }
          }
        }
      }
    }
  ]
}
```

**Delete a Sheet Request:**

```json
{
  "requests": [
    {
      "deleteSheet": {
        "sheetId": 12345
      }
    }
  ]
}
```

**Delete Rows/Columns Request:**

```json
{
  "requests": [
    {
      "deleteDimension": {
        "range": {
          "sheetId": 0,
          "dimension": "ROWS",
          "startIndex": 5,
          "endIndex": 10
        }
      }
    }
  ]
}
```

**Response (200 OK):**

```json
{
  "spreadsheetId": "1BxiMVs0XRA...",
  "replies": [
    {
      "addSheet": {
        "properties": {
          "sheetId": 67890,
          "title": "March Expenses",
          "index": 2,
          "sheetType": "GRID",
          "gridProperties": {
            "rowCount": 1000,
            "columnCount": 26
          }
        }
      }
    }
  ]
}
```

**Error Codes (All Sheets Endpoints):**

| Code | Description |
|---|---|
| 400 | Invalid range, bad A1 notation, or invalid request body |
| 401 | Authentication failure |
| 403 | Permission denied or quota exceeded |
| 404 | Spreadsheet or sheet not found |
| 429 | Rate limit exceeded |
| 500 | Internal server error |

### 2.6 Webhooks / Real-time

Same as Google Docs: use Google Drive API `files.watch` endpoint. No native Sheets-level push notifications exist.

```
POST https://www.googleapis.com/drive/v3/files/{spreadsheetId}/watch
```

The notification only indicates that the file changed -- Esmer must then call the Sheets API to determine what changed by comparing cached data with fresh reads.

### 2.7 Error Handling

| HTTP Code | Meaning | Esmer Action |
|---|---|---|
| 400 | Invalid range or request | Parse error message for range issues; fix and retry if auto-correctable |
| 401 | Token expired | Refresh token, retry |
| 403 | Permission or quota | Check scopes; if `RATE_LIMIT_EXCEEDED`, backoff |
| 404 | Sheet/spreadsheet not found | Remove from cache, notify user |
| 429 | Too many requests | Exponential backoff starting at 1s |
| 500/503 | Server errors | Retry up to 3 times with backoff |

**Common Issue -- Column names changed:** If the spreadsheet's header row changes after Esmer cached it, field mapping will break. Esmer should re-read the header row (row 1) before each write operation, or cache headers with a short TTL (2 minutes).

### 2.8 Mobile-Specific Notes

- **A1 Notation Caching:** Cache the header row mapping (column letter to column name) locally. Rebuild on first access and after any write failure.
- **Append-First Pattern:** For data entry, always use the `values:append` endpoint rather than calculating the next empty row manually. This avoids race conditions with concurrent editors.
- **valueInputOption:** Use `USER_ENTERED` for user-facing data so formulas and dates are parsed naturally. Use `RAW` for machine-generated data to preserve exact values.
- **Partial Reads:** Use the `fields` query parameter and specific ranges to minimize payload size. Instead of reading an entire sheet, read only the columns Esmer needs.
- **Array Data:** When inserting array data, convert it to key-value JSON pairs first. Use a data transformation step before calling the append endpoint.
- **Offline Append Queue:** Queue rows for appending while offline. On reconnect, use a single `values:append` call with all queued rows as a 2D array.

---

## 3. Microsoft Excel 365 (Microsoft Graph API)

### 3.1 Service Overview

Esmer uses Microsoft Excel 365 via the Microsoft Graph API for users in Microsoft-centric organizations: reading workbook data, adding rows to structured tables, retrieving worksheet contents, and managing workbooks. Key mobile workflows include enterprise data entry, reading shared team spreadsheets, and adding structured records to Excel tables stored in OneDrive or SharePoint.

### 3.2 Authentication

**Method:** OAuth 2.0 via Microsoft Identity Platform

**OAuth 2.0 Flow:** Authorization Code with PKCE

**Authorization Endpoint:** `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize`
**Token Endpoint:** `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token`

Use `common` for the `{tenant}` value to support multi-tenant (personal + organizational accounts).

| Scope | Description | Required For |
|---|---|---|
| `Files.ReadWrite` | Read/write access to user's files | All Excel operations on OneDrive files |
| `Files.ReadWrite.All` | Read/write all files user can access | Shared and team files |
| `Sites.ReadWrite.All` | Read/write SharePoint sites | Excel files in SharePoint |
| `offline_access` | Obtain refresh tokens | Persistent sessions |
| `openid` | OpenID Connect sign-in | User identity |

**Government Cloud Endpoints:**

| Environment | Authorization URL | Graph Base URL |
|---|---|---|
| Global | `login.microsoftonline.com` | `graph.microsoft.com` |
| US Government (GCC) | `login.microsoftonline.us` | `graph.microsoft.us` |
| US Government DOD | `login.microsoftonline.us` | `dod-graph.microsoft.us` |
| China (21Vianet) | `login.chinacloudapi.cn` | `microsoftgraph.chinacloudapi.cn` |

**Custom Scopes:** Microsoft Excel credentials support custom scopes for granular permission control.

**Recommendation for Esmer:** Request `Files.ReadWrite` + `offline_access` as minimum. Add `Sites.ReadWrite.All` only when the user needs SharePoint-hosted Excel files. Store tokens in platform-specific secure storage.

### 3.3 Base URL

```
https://graph.microsoft.com/v1.0
```

Excel resources are accessed under the DriveItem path:

```
/me/drive/items/{item-id}/workbook
```

Or via path:

```
/me/drive/root:/{file-path}:/workbook
```

### 3.4 Rate Limits

| Quota | Limit | Window |
|---|---|---|
| Application + user combined | 10,000 requests per 10 minutes | 10 minutes |
| Per-app per-user | Subset of above | 10 minutes |
| Concurrent requests per app per user | 4 concurrent requests | N/A |
| Excel-specific throttle | Additional throttle on long-running operations | Variable |

**Note:** Microsoft Graph does not publish exact per-endpoint limits for Excel. The service may return `429 Too Many Requests` with a `Retry-After` header indicating the number of seconds to wait.

**Esmer Strategy:** Respect `Retry-After` headers precisely. Limit concurrent Excel API calls to 4 per user. Prefer batch JSON requests for multi-step operations.

### 3.5 API Endpoints

#### 3.5.1 Workbook Resource

##### Get Workbook Metadata

```
GET /me/drive/items/{item-id}/workbook
```

**Description:** Retrieves the workbook resource including its application and session properties.

**Response (200 OK):**

```json
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#workbook",
  "application": {
    "name": "Microsoft Excel"
  }
}
```

##### Create a Workbook Session

```
POST /me/drive/items/{item-id}/workbook/createSession
```

**Description:** Creates a persistent session for batching multiple operations. Sessions improve performance and ensure consistency.

**Request:**

```json
{
  "persistChanges": true
}
```

**Response (201 Created):**

```json
{
  "id": "cluster=ABC123&session=DEF456",
  "persistChanges": true
}
```

Use the session ID in subsequent requests via the `workbook-session-id` header.

##### Close Session

```
POST /me/drive/items/{item-id}/workbook/closeSession
```

**Headers:** `workbook-session-id: {session-id}`

**Response:** `204 No Content`

##### List Workbooks (via Drive)

```
GET /me/drive/root/search(q='.xlsx')
```

**Description:** Search for Excel workbooks in the user's OneDrive.

**Response (200 OK):**

```json
{
  "value": [
    {
      "id": "01BYE5RZ...",
      "name": "Budget 2026.xlsx",
      "webUrl": "https://onedrive.live.com/...",
      "file": {
        "mimeType": "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
      },
      "size": 45678,
      "lastModifiedDateTime": "2026-02-08T14:30:00Z"
    }
  ]
}
```

#### 3.5.2 Worksheet Resource

##### List All Worksheets

```
GET /me/drive/items/{item-id}/workbook/worksheets
```

**Description:** Returns all worksheets in a workbook.

**Response (200 OK):**

```json
{
  "value": [
    {
      "id": "{sheet-id-1}",
      "name": "Sheet1",
      "position": 0,
      "visibility": "Visible"
    },
    {
      "id": "{sheet-id-2}",
      "name": "Summary",
      "position": 1,
      "visibility": "Visible"
    }
  ]
}
```

##### Add a Worksheet

```
POST /me/drive/items/{item-id}/workbook/worksheets
```

**Description:** Adds a new worksheet to the workbook.

**Request:**

```json
{
  "name": "March Data"
}
```

**Response (201 Created):**

```json
{
  "id": "{new-sheet-id}",
  "name": "March Data",
  "position": 2,
  "visibility": "Visible"
}
```

##### Get Worksheet Content (Used Range)

```
GET /me/drive/items/{item-id}/workbook/worksheets/{sheet-id-or-name}/usedRange
```

**Description:** Returns the range of cells that contain data.

**Response (200 OK):**

```json
{
  "address": "Sheet1!A1:E5",
  "addressLocal": "Sheet1!A1:E5",
  "cellCount": 25,
  "columnCount": 5,
  "rowCount": 5,
  "values": [
    ["Date", "Category", "Description", "Amount", "Status"],
    ["2026-02-01", "Marketing", "Ad spend", 500, "Approved"],
    ["2026-02-03", "Engineering", "Cloud hosting", 1200, "Approved"],
    ["2026-02-05", "Sales", "Client gifts", 150, "Pending"],
    ["2026-02-07", "Marketing", "Conference", 2000, "Approved"]
  ],
  "formulas": [
    ["Date", "Category", "Description", "Amount", "Status"],
    ["2026-02-01", "Marketing", "Ad spend", 500, "Approved"],
    ["2026-02-03", "Engineering", "Cloud hosting", 1200, "Approved"],
    ["2026-02-05", "Sales", "Client gifts", 150, "Pending"],
    ["2026-02-07", "Marketing", "Conference", 2000, "Approved"]
  ],
  "numberFormat": [
    ["General", "General", "General", "General", "General"],
    ["General", "General", "General", "#,##0.00", "General"],
    ["General", "General", "General", "#,##0.00", "General"],
    ["General", "General", "General", "#,##0.00", "General"],
    ["General", "General", "General", "#,##0.00", "General"]
  ],
  "text": [
    ["Date", "Category", "Description", "Amount", "Status"],
    ["2026-02-01", "Marketing", "Ad spend", "500.00", "Approved"],
    ["2026-02-03", "Engineering", "Cloud hosting", "1200.00", "Approved"],
    ["2026-02-05", "Sales", "Client gifts", "150.00", "Pending"],
    ["2026-02-07", "Marketing", "Conference", "2000.00", "Approved"]
  ]
}
```

##### Get a Specific Range

```
GET /me/drive/items/{item-id}/workbook/worksheets/{sheet-id-or-name}/range(address='A1:E5')
```

**Description:** Returns values for a specific cell range.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `item-id` | path | string | Yes | Drive item ID of the workbook |
| `sheet-id-or-name` | path | string | Yes | Worksheet ID or name |
| `address` | path | string | Yes | Cell range in A1 notation |

**Response (200 OK):** Same structure as `usedRange` above.

##### Update a Range

```
PATCH /me/drive/items/{item-id}/workbook/worksheets/{sheet-id-or-name}/range(address='A6:E6')
```

**Description:** Updates cell values in a specified range.

**Request:**

```json
{
  "values": [
    ["2026-02-09", "HR", "Team building", 300, "Pending"]
  ]
}
```

**Response (200 OK):**

```json
{
  "address": "Sheet1!A6:E6",
  "cellCount": 5,
  "columnCount": 5,
  "rowCount": 1,
  "values": [
    ["2026-02-09", "HR", "Team building", 300, "Pending"]
  ]
}
```

#### 3.5.3 Table Resource

##### List Tables in a Worksheet

```
GET /me/drive/items/{item-id}/workbook/worksheets/{sheet-id-or-name}/tables
```

**Response (200 OK):**

```json
{
  "value": [
    {
      "id": "{table-id}",
      "name": "ExpenseTable",
      "showHeaders": true,
      "showTotals": false,
      "style": "TableStyleMedium2"
    }
  ]
}
```

##### Get Table Columns

```
GET /me/drive/items/{item-id}/workbook/tables/{table-id-or-name}/columns
```

**Response (200 OK):**

```json
{
  "value": [
    { "id": 1, "name": "Date", "index": 0 },
    { "id": 2, "name": "Category", "index": 1 },
    { "id": 3, "name": "Description", "index": 2 },
    { "id": 4, "name": "Amount", "index": 3 },
    { "id": 5, "name": "Status", "index": 4 }
  ]
}
```

##### Get Table Rows

```
GET /me/drive/items/{item-id}/workbook/tables/{table-id-or-name}/rows
```

**Response (200 OK):**

```json
{
  "value": [
    {
      "index": 0,
      "values": [["2026-02-01", "Marketing", "Ad spend", 500, "Approved"]]
    },
    {
      "index": 1,
      "values": [["2026-02-03", "Engineering", "Cloud hosting", 1200, "Approved"]]
    }
  ]
}
```

##### Add Rows to a Table

```
POST /me/drive/items/{item-id}/workbook/tables/{table-id-or-name}/rows
```

**Description:** Appends one or more rows to the end of a table.

**Request:**

```json
{
  "values": [
    ["2026-02-09", "HR", "Team building", 300, "Pending"],
    ["2026-02-09", "IT", "Software license", 50, "Approved"]
  ]
}
```

**Response (201 Created):**

```json
{
  "index": 4,
  "values": [
    ["2026-02-09", "HR", "Team building", 300, "Pending"],
    ["2026-02-09", "IT", "Software license", 50, "Approved"]
  ]
}
```

##### Look Up a Row by Column Value

```
GET /me/drive/items/{item-id}/workbook/tables/{table-id-or-name}/rows
```

Then filter client-side. Alternatively, use the worksheet `range` endpoint with a filter function:

```
POST /me/drive/items/{item-id}/workbook/functions/vlookup
```

**Request:**

```json
{
  "lookupValue": "Engineering",
  "tableArray": {
    "address": "Sheet1!B2:E10"
  },
  "colIndexNum": 3,
  "rangeLookup": false
}
```

**Response (200 OK):**

```json
{
  "value": 1200
}
```

**Error Codes (All Excel Endpoints):**

| Code | Description |
|---|---|
| 400 | Bad request (invalid range, malformed body) |
| 401 | Authentication failure |
| 403 | Insufficient permissions |
| 404 | Workbook, worksheet, or table not found |
| 409 | Conflict (concurrent edit session) |
| 429 | Throttled -- check `Retry-After` header |
| 500 | Internal server error |
| 503 | Service unavailable |
| 504 | Gateway timeout (large workbook operations) |

### 3.6 Webhooks / Real-time

Microsoft Graph supports change notifications via subscriptions.

##### Create Subscription

```
POST https://graph.microsoft.com/v1.0/subscriptions
```

**Request:**

```json
{
  "changeType": "updated",
  "notificationUrl": "https://api.esmer.app/webhooks/microsoft-graph",
  "resource": "/me/drive/items/{item-id}",
  "expirationDateTime": "2026-02-10T12:00:00Z",
  "clientState": "esmer-secret-state-value"
}
```

**Response (201 Created):**

```json
{
  "id": "subscription-id-uuid",
  "resource": "/me/drive/items/{item-id}",
  "changeType": "updated",
  "notificationUrl": "https://api.esmer.app/webhooks/microsoft-graph",
  "expirationDateTime": "2026-02-10T12:00:00Z",
  "clientState": "esmer-secret-state-value"
}
```

**Subscription limits:** Maximum expiration 3 days for Drive items. Esmer must renew subscriptions before they expire.

**Notification Payload:**

```json
{
  "value": [
    {
      "subscriptionId": "subscription-id-uuid",
      "clientState": "esmer-secret-state-value",
      "changeType": "updated",
      "resource": "me/drive/items/{item-id}",
      "resourceData": {
        "@odata.type": "#Microsoft.Graph.DriveItem",
        "@odata.id": "me/drive/items/{item-id}",
        "id": "{item-id}"
      },
      "tenantId": "tenant-uuid"
    }
  ]
}
```

### 3.7 Error Handling

| HTTP Code | Meaning | Esmer Action |
|---|---|---|
| 400 | Bad request | Parse `error.message` for details; fix request |
| 401 | Token expired or invalid | Refresh token and retry once |
| 403 | Insufficient permissions | Prompt user to re-authorize with required scopes |
| 404 | Resource not found | Check item-id; notify user if file was deleted |
| 409 | Edit conflict | Close session, create new session, retry |
| 429 | Throttled | Read `Retry-After` header (seconds), wait, then retry |
| 500 | Internal error | Retry with backoff up to 3 times |
| 503 | Service unavailable | Retry with backoff up to 5 times |
| 504 | Gateway timeout | For large workbooks, break operations into smaller chunks |

**Session Management:** Always close sessions (`closeSession`) when done. Orphaned sessions can cause `409 Conflict` errors for subsequent requests against the same workbook.

### 3.8 Mobile-Specific Notes

- **Session Strategy:** Create a session at the start of a multi-step operation and close it when done. This reduces latency by keeping the workbook open server-side. Set `persistChanges: true` so changes are saved even if the session is not cleanly closed (e.g., app goes to background).
- **Large Workbooks:** Avoid requesting the full `usedRange` for large worksheets. Instead, use specific `range(address='...')` calls targeting only the data Esmer needs.
- **OneDrive vs SharePoint:** Files in OneDrive use `/me/drive/items/{id}`. Files in SharePoint use `/sites/{site-id}/drive/items/{id}`. Detect the source from the file's `webUrl` or `parentReference.driveType`.
- **Offline:** Cache the last-known table columns and recent rows. Queue new row additions offline and POST them in bulk on reconnect.
- **Government Cloud:** Esmer must detect the user's tenant type during OAuth and route API calls to the correct base URL (`graph.microsoft.com` vs `graph.microsoft.us`).

---

## 4. Google Slides (Google Slides API v1)

### 4.1 Service Overview

Esmer uses Google Slides for presentation automation on behalf of mobile users: creating presentations from templates, reading slide content for summarization, performing text replacement for dynamic report generation, and retrieving slide thumbnails for preview. Key mobile workflows include generating status-update decks, filling template presentations with data from other Esmer integrations, and previewing presentation content without opening the full editor.

### 4.2 Authentication

**Method:** OAuth 2.0 (recommended) or Google Service Account

**OAuth 2.0 Flow:** Authorization Code with PKCE

| Scope | Description | Required For |
|---|---|---|
| `https://www.googleapis.com/auth/presentations` | Full read/write access to Slides | Create, update presentations |
| `https://www.googleapis.com/auth/presentations.readonly` | Read-only access | Get presentations and pages |
| `https://www.googleapis.com/auth/drive` | Full Drive access | File management |
| `https://www.googleapis.com/auth/drive.file` | App-created/opened files only | Minimum file scope |

**Recommendation for Esmer:** Use `presentations` + `drive.file`. Add `drive.readonly` if Esmer needs to list presentations the user did not create through the app.

### 4.3 Base URL

```
https://slides.googleapis.com/v1
```

### 4.4 Rate Limits

| Quota | Limit | Window |
|---|---|---|
| Read requests | 300 per minute per project | Per minute |
| Write requests | 60 per minute per user per project | Per minute |

Same Google Workspace API quota framework as Docs and Sheets.

### 4.5 API Endpoints

#### 4.5.1 Presentations Resource

##### Create a Presentation

```
POST /v1/presentations
```

**Description:** Creates a new blank presentation.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `title` | body | string | Yes | Title of the presentation |

**Request:**

```json
{
  "title": "Q1 2026 Status Report - Generated by Esmer"
}
```

**Response (200 OK):**

```json
{
  "presentationId": "1abc2DEF3ghi4JKL5mno6PQR7stu8VWX",
  "pageSize": {
    "width": { "magnitude": 9144000, "unit": "EMU" },
    "height": { "magnitude": 5143500, "unit": "EMU" }
  },
  "slides": [
    {
      "objectId": "p",
      "pageType": "SLIDE",
      "pageElements": [],
      "slideProperties": {
        "layoutObjectId": "layout-id",
        "masterObjectId": "master-id"
      }
    }
  ],
  "title": "Q1 2026 Status Report - Generated by Esmer",
  "masters": [...],
  "layouts": [...],
  "locale": "en"
}
```

##### Get a Presentation

```
GET /v1/presentations/{presentationId}
```

**Description:** Retrieves the full presentation including all slides, masters, and layouts.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `presentationId` | path | string | Yes | The presentation ID |

**Response (200 OK):** Full presentation object including all slides, page elements, text content, images, and styling.

##### Get Slides (List)

```
GET /v1/presentations/{presentationId}
```

**Description:** Returns all slides in the presentation. Use the `fields` parameter to limit the response.

**Request:**

```
GET /v1/presentations/1abc2DEF...?fields=slides(objectId,pageElements)
```

**Response (200 OK):**

```json
{
  "slides": [
    {
      "objectId": "slide-page-1",
      "pageElements": [
        {
          "objectId": "title-element-1",
          "size": {
            "width": { "magnitude": 8229600, "unit": "EMU" },
            "height": { "magnitude": 1143000, "unit": "EMU" }
          },
          "transform": {
            "scaleX": 1.0,
            "scaleY": 1.0,
            "translateX": 457200,
            "translateY": 274638,
            "unit": "EMU"
          },
          "shape": {
            "shapeType": "TEXT_BOX",
            "text": {
              "textElements": [
                {
                  "startIndex": 0,
                  "endIndex": 24,
                  "textRun": {
                    "content": "Q1 2026 Status Report\n",
                    "style": {
                      "fontSize": { "magnitude": 28, "unit": "PT" },
                      "bold": true
                    }
                  }
                }
              ]
            }
          }
        }
      ]
    }
  ]
}
```

#### 4.5.2 Pages Resource

##### Get a Page

```
GET /v1/presentations/{presentationId}/pages/{pageObjectId}
```

**Description:** Returns the details of a single page (slide, layout, or master).

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `presentationId` | path | string | Yes | The presentation ID |
| `pageObjectId` | path | string | Yes | The page object ID |

**Response (200 OK):**

```json
{
  "objectId": "slide-page-1",
  "pageType": "SLIDE",
  "pageElements": [...],
  "slideProperties": {
    "layoutObjectId": "layout-1",
    "masterObjectId": "master-1"
  },
  "pageProperties": {
    "pageBackgroundFill": {
      "solidFill": {
        "color": {
          "rgbColor": {
            "red": 1.0,
            "green": 1.0,
            "blue": 1.0
          }
        }
      }
    }
  }
}
```

##### Get a Page Thumbnail

```
GET /v1/presentations/{presentationId}/pages/{pageObjectId}/thumbnail
```

**Description:** Returns a thumbnail image URL for a specific slide. Essential for mobile preview.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `presentationId` | path | string | Yes | The presentation ID |
| `pageObjectId` | path | string | Yes | The page object ID |
| `thumbnailProperties.mimeType` | query | enum | No | `PNG` (default) or `JPEG` (not supported) |
| `thumbnailProperties.thumbnailSize` | query | enum | No | `SMALL` (200px), `MEDIUM` (800px), `LARGE` (1600px) |

**Request:**

```
GET /v1/presentations/1abc2DEF.../pages/slide-page-1/thumbnail?thumbnailProperties.thumbnailSize=MEDIUM
```

**Response (200 OK):**

```json
{
  "width": 800,
  "height": 450,
  "contentUrl": "https://lh3.googleusercontent.com/..."
}
```

The `contentUrl` is a temporary URL (valid for ~30 minutes). Esmer should download and cache the thumbnail locally.

#### 4.5.3 Batch Update (Write Operations)

##### Batch Update a Presentation

```
POST /v1/presentations/{presentationId}:batchUpdate
```

**Description:** Applies one or more mutations to a presentation. All modifications go through this single endpoint.

**Available Request Types:**

| Request Type | Description |
|---|---|
| `replaceAllText` | Find and replace text across all slides |
| `createSlide` | Add a new slide |
| `deleteObject` | Delete a page element or slide |
| `createShape` | Create a shape (text box, rectangle, etc.) |
| `insertText` | Insert text into a shape |
| `deleteText` | Delete text from a shape |
| `updateTextStyle` | Update text formatting |
| `updateParagraphStyle` | Update paragraph alignment, spacing |
| `createImage` | Insert an image |
| `replaceAllShapesWithImage` | Replace placeholder shapes with images |
| `replaceAllShapesWithSheetsChart` | Embed a Google Sheets chart |
| `createTable` | Create a table on a slide |
| `insertTableRows` | Add rows to a table |
| `insertTableColumns` | Add columns to a table |
| `deleteTableRow` | Remove a table row |
| `deleteTableColumn` | Remove a table column |
| `updateTableCellProperties` | Update cell borders, fill |
| `mergeTableCells` | Merge cells |
| `unmergeTableCells` | Unmerge cells |
| `createVideo` | Embed a video |
| `updatePageProperties` | Update background, transitions |
| `updateSlideProperties` | Update slide layout reference |
| `duplicateObject` | Duplicate a slide or element |
| `groupObjects` | Group elements |
| `ungroupObjects` | Ungroup elements |
| `updatePageElementTransform` | Move/resize elements |
| `updateShapeProperties` | Update fill, outline, shadow |
| `updateImageProperties` | Update image crop, recolor |
| `createParagraphBullets` | Add bullets to text |
| `updateTableRowProperties` | Update row height |
| `updateTableColumnProperties` | Update column width |
| `updateTableBorderProperties` | Update cell borders |
| `updateLineProperties` | Update line/connector styling |

**Request Example -- Replace All Text (Template Fill):**

```json
{
  "requests": [
    {
      "replaceAllText": {
        "containsText": {
          "text": "{{REPORT_TITLE}}",
          "matchCase": true
        },
        "replaceText": "Q1 2026 Engineering Status"
      }
    },
    {
      "replaceAllText": {
        "containsText": {
          "text": "{{DATE}}",
          "matchCase": true
        },
        "replaceText": "February 9, 2026"
      }
    },
    {
      "replaceAllText": {
        "containsText": {
          "text": "{{AUTHOR}}",
          "matchCase": true
        },
        "replaceText": "Generated by Esmer AI"
      }
    }
  ]
}
```

**Request Example -- Create a New Slide:**

```json
{
  "requests": [
    {
      "createSlide": {
        "objectId": "new-slide-001",
        "insertionIndex": 1,
        "slideLayoutReference": {
          "predefinedLayout": "TITLE_AND_BODY"
        },
        "placeholderIdMappings": [
          {
            "layoutPlaceholder": {
              "type": "TITLE",
              "index": 0
            },
            "objectId": "new-title-001"
          },
          {
            "layoutPlaceholder": {
              "type": "BODY",
              "index": 0
            },
            "objectId": "new-body-001"
          }
        ]
      }
    },
    {
      "insertText": {
        "objectId": "new-title-001",
        "text": "Key Achievements",
        "insertionIndex": 0
      }
    },
    {
      "insertText": {
        "objectId": "new-body-001",
        "text": "Launched mobile app v2.0\nReduced API latency by 40%\nOnboarded 500 new users",
        "insertionIndex": 0
      }
    }
  ]
}
```

**Response (200 OK):**

```json
{
  "presentationId": "1abc2DEF3ghi4JKL5mno6PQR7stu8VWX",
  "replies": [
    {
      "replaceAllText": {
        "occurrencesChanged": 3
      }
    },
    {
      "replaceAllText": {
        "occurrencesChanged": 2
      }
    },
    {
      "replaceAllText": {
        "occurrencesChanged": 1
      }
    }
  ]
}
```

**Error Codes:**

| Code | Description |
|---|---|
| 400 | Invalid request (bad object ID, invalid layout reference) |
| 401 | Authentication failure |
| 403 | Insufficient permissions or quota exceeded |
| 404 | Presentation or page not found |
| 429 | Rate limit exceeded |

### 4.6 Webhooks / Real-time

Same as Google Docs and Sheets: use Google Drive API `files.watch` for change notifications. No native Slides-level push notifications.

### 4.7 Error Handling

Same error handling strategy as Google Docs (Section 1.7). Key difference: Slides uses EMU (English Metric Units) for positioning. Invalid EMU values in `updatePageElementTransform` requests will cause `400` errors.

**EMU Reference:** 1 inch = 914400 EMU. 1 point = 12700 EMU.

### 4.8 Mobile-Specific Notes

- **Thumbnail Caching:** Always download and cache slide thumbnails locally. The `contentUrl` from the thumbnail endpoint expires in approximately 30 minutes. Use `MEDIUM` (800px) size for phone displays and `SMALL` (200px) for list/grid views.
- **Template Strategy:** The most common mobile use case is template filling via `replaceAllText`. Pre-define templates with `{{PLACEHOLDER}}` tokens. Esmer can fill an entire presentation with a single `batchUpdate` call containing multiple `replaceAllText` requests.
- **Read-Only Priority:** Most mobile users will view presentations rather than create them from scratch. Optimize for fast slide listing and thumbnail retrieval. Pre-fetch thumbnails for the first 5 slides.
- **EMU Math:** Abstract EMU calculations away from mobile users. Provide preset positions (centered, top-left, full-width) that Esmer translates to EMU values.
- **Offline:** Cache presentation metadata and thumbnails. Text replacement requests can be queued offline and applied on reconnect.

---

## 5. Airtable (Airtable REST API)

### 5.1 Service Overview

Esmer uses Airtable as a flexible, user-friendly structured database for knowledge management: creating and reading records, updating existing entries, listing records with filters and sorting, and managing table structures. Key mobile workflows include CRM data entry, project task tracking, inventory management, content calendars, and any scenario where users maintain structured data with rich field types (attachments, linked records, formulas, rollups).

### 5.2 Authentication

**Method:** Personal Access Token (PAT) -- recommended, or OAuth 2.0

#### Personal Access Token (PAT)

| Parameter | Value |
|---|---|
| Header | `Authorization: Bearer {personal-access-token}` |
| Token Creation | https://airtable.com/create/tokens |

**Required Scopes for PAT:**

| Scope | Description | Required For |
|---|---|---|
| `data.records:read` | Read records from bases | List, get records |
| `data.records:write` | Create, update, delete records | Write operations |
| `schema.bases:read` | Read base schema (tables, fields, views) | List tables, fields, views |
| `schema.bases:write` | Modify base schema | Create/modify tables and fields |
| `webhook:manage` | Create and manage webhooks | Real-time change notifications |

**Access Configuration:** When creating a PAT, you must specify which bases the token can access: a single base, multiple bases, all bases in a workspace, or all bases across all workspaces.

#### OAuth 2.0

**Authorization Endpoint:** `https://airtable.com/oauth2/v1/authorize`
**Token Endpoint:** `https://airtable.com/oauth2/v1/token`

| Parameter | Value |
|---|---|
| `response_type` | `code` |
| `client_id` | Your registered OAuth integration client ID |
| `redirect_uri` | Your registered redirect URI |
| `scope` | Space-separated scopes (same as PAT scopes above) |
| `state` | CSRF protection token |
| `code_challenge` | PKCE code challenge (S256) |
| `code_challenge_method` | `S256` |

**Token Exchange:**

```
POST https://airtable.com/oauth2/v1/token
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(client_id:client_secret)

grant_type=authorization_code&code={code}&redirect_uri={redirect_uri}&code_verifier={code_verifier}
```

**Token Refresh:**

```
POST https://airtable.com/oauth2/v1/token
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(client_id:client_secret)

grant_type=refresh_token&refresh_token={refresh_token}
```

Access tokens expire after 2 hours. Refresh tokens expire after 60 days (extended on each use).

**Recommendation for Esmer:** Use OAuth 2.0 with PKCE for mobile app distribution. Use PAT for internal/developer use. OAuth provides better user experience (no manual token copying) and supports token revocation.

### 5.3 Base URL

```
https://api.airtable.com/v0
```

All record operations are scoped to a base and table:

```
https://api.airtable.com/v0/{baseId}/{tableIdOrName}
```

### 5.4 Rate Limits

| Quota | Limit | Window |
|---|---|---|
| Per base per token | 5 requests per second | Per second |
| Across all bases per token | 50 requests per second | Per second |
| Batch record limit | 10 records per create/update/delete request | Per request |
| Max records per base | 100,000 | N/A |
| List page size | 100 records per page (max) | Per request |

**On Rate Limit (429):** Airtable requires a 30-second wait before resuming requests.

**Esmer Strategy:** Implement a per-base request queue with 200ms minimum spacing between requests. Batch record operations in groups of 10. Implement 30-second backoff on 429 responses.

### 5.5 API Endpoints

#### 5.5.1 Records Resource

##### List Records

```
GET /v0/{baseId}/{tableIdOrName}
```

**Description:** Returns a paginated list of records from a table.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `baseId` | path | string | Yes | The base ID (starts with `app`) |
| `tableIdOrName` | path | string | Yes | Table ID (starts with `tbl`) or table name (URL-encoded) |
| `fields[]` | query | string[] | No | Only return specific fields |
| `filterByFormula` | query | string | No | Airtable formula to filter records |
| `maxRecords` | query | integer | No | Max total records to return |
| `pageSize` | query | integer | No | Records per page (max 100, default 100) |
| `sort[0][field]` | query | string | No | Field name to sort by |
| `sort[0][direction]` | query | string | No | `asc` or `desc` |
| `view` | query | string | No | View name or ID to use (applies its filters/sorts) |
| `offset` | query | string | No | Pagination cursor from previous response |
| `cellFormat` | query | enum | No | `json` (default) or `string` |
| `timeZone` | query | string | No | Timezone for date formatting |
| `userLocale` | query | string | No | Locale for date formatting |
| `returnFieldsByFieldId` | query | boolean | No | Use field IDs instead of names |

**Request:**

```
GET /v0/appABC123/Tasks?filterByFormula=AND({Status}='In Progress',{Assignee}='harjas')&sort[0][field]=Due Date&sort[0][direction]=asc&pageSize=20
```

**Response (200 OK):**

```json
{
  "records": [
    {
      "id": "recABC123",
      "createdTime": "2026-02-01T10:00:00.000Z",
      "fields": {
        "Task Name": "Review API spec",
        "Status": "In Progress",
        "Assignee": "harjas",
        "Due Date": "2026-02-10",
        "Priority": "High",
        "Tags": ["documentation", "api"],
        "Estimated Hours": 4,
        "Project": ["recPROJ001"]
      }
    },
    {
      "id": "recDEF456",
      "createdTime": "2026-02-03T14:30:00.000Z",
      "fields": {
        "Task Name": "Update mobile UI",
        "Status": "In Progress",
        "Assignee": "harjas",
        "Due Date": "2026-02-12",
        "Priority": "Medium",
        "Tags": ["mobile", "ui"],
        "Estimated Hours": 8,
        "Project": ["recPROJ001"]
      }
    }
  ],
  "offset": "itr8UpQ3z7NEXT_PAGE_CURSOR"
}
```

**Pagination:** If `offset` is present in the response, more records exist. Pass it as a query parameter in the next request. Continue until no `offset` is returned.

**Filter Formula Examples:**

| Use Case | Formula |
|---|---|
| Exact match | `{Status}='Done'` |
| Not equal | `NOT({Status}='Done')` |
| Contains text | `FIND('urgent', LOWER({Description})) > 0` |
| Date comparison | `IS_AFTER({Due Date}, TODAY())` |
| Multiple conditions | `AND({Status}='Active', {Priority}='High')` |
| OR conditions | `OR({Status}='New', {Status}='In Progress')` |
| Empty field | `{Assignee}=BLANK()` |
| Not empty | `NOT({Assignee}=BLANK())` |
| Numeric comparison | `{Amount} > 100` |
| Record from linked table | `FIND('recABC123', ARRAYJOIN(RECORD_ID())) > 0` |

##### Get a Single Record

```
GET /v0/{baseId}/{tableIdOrName}/{recordId}
```

**Description:** Returns a single record by its ID.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `recordId` | path | string | Yes | The record ID (starts with `rec`) |
| `returnFieldsByFieldId` | query | boolean | No | Use field IDs instead of names |

**Response (200 OK):**

```json
{
  "id": "recABC123",
  "createdTime": "2026-02-01T10:00:00.000Z",
  "fields": {
    "Task Name": "Review API spec",
    "Status": "In Progress",
    "Assignee": "harjas",
    "Due Date": "2026-02-10",
    "Priority": "High",
    "Tags": ["documentation", "api"],
    "Estimated Hours": 4
  }
}
```

##### Create Records

```
POST /v0/{baseId}/{tableIdOrName}
```

**Description:** Creates one or more new records. Maximum 10 records per request.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `records` | body | array | Yes | Array of record objects (max 10) |
| `typecast` | body | boolean | No | If `true`, auto-convert string values to proper field types |
| `returnFieldsByFieldId` | body | boolean | No | Use field IDs in response |

**Request:**

```json
{
  "records": [
    {
      "fields": {
        "Task Name": "Write integration tests",
        "Status": "New",
        "Assignee": "harjas",
        "Due Date": "2026-02-15",
        "Priority": "High",
        "Tags": ["testing", "api"],
        "Estimated Hours": 6
      }
    },
    {
      "fields": {
        "Task Name": "Deploy staging environment",
        "Status": "New",
        "Assignee": "harjas",
        "Due Date": "2026-02-14",
        "Priority": "Medium",
        "Tags": ["devops"],
        "Estimated Hours": 2
      }
    }
  ],
  "typecast": true
}
```

**Response (200 OK):**

```json
{
  "records": [
    {
      "id": "recNEW001",
      "createdTime": "2026-02-09T12:00:00.000Z",
      "fields": {
        "Task Name": "Write integration tests",
        "Status": "New",
        "Assignee": "harjas",
        "Due Date": "2026-02-15",
        "Priority": "High",
        "Tags": ["testing", "api"],
        "Estimated Hours": 6
      }
    },
    {
      "id": "recNEW002",
      "createdTime": "2026-02-09T12:00:00.000Z",
      "fields": {
        "Task Name": "Deploy staging environment",
        "Status": "New",
        "Assignee": "harjas",
        "Due Date": "2026-02-14",
        "Priority": "Medium",
        "Tags": ["devops"],
        "Estimated Hours": 2
      }
    }
  ]
}
```

##### Update Records (PATCH -- partial update)

```
PATCH /v0/{baseId}/{tableIdOrName}
```

**Description:** Updates one or more existing records. Only specified fields are updated; unspecified fields remain unchanged. Maximum 10 records per request.

**Request:**

```json
{
  "records": [
    {
      "id": "recABC123",
      "fields": {
        "Status": "Done",
        "Actual Hours": 3.5
      }
    },
    {
      "id": "recDEF456",
      "fields": {
        "Status": "In Review",
        "Priority": "High"
      }
    }
  ],
  "typecast": true
}
```

**Response (200 OK):**

```json
{
  "records": [
    {
      "id": "recABC123",
      "createdTime": "2026-02-01T10:00:00.000Z",
      "fields": {
        "Task Name": "Review API spec",
        "Status": "Done",
        "Assignee": "harjas",
        "Due Date": "2026-02-10",
        "Priority": "High",
        "Tags": ["documentation", "api"],
        "Estimated Hours": 4,
        "Actual Hours": 3.5
      }
    },
    {
      "id": "recDEF456",
      "createdTime": "2026-02-03T14:30:00.000Z",
      "fields": {
        "Task Name": "Update mobile UI",
        "Status": "In Review",
        "Assignee": "harjas",
        "Due Date": "2026-02-12",
        "Priority": "High",
        "Tags": ["mobile", "ui"],
        "Estimated Hours": 8
      }
    }
  ]
}
```

##### Update Records (PUT -- destructive update)

```
PUT /v0/{baseId}/{tableIdOrName}
```

**Description:** Replaces all fields of existing records. Fields not included in the request will be cleared. Maximum 10 records per request.

Same request/response format as PATCH, but unspecified fields are set to empty/null.

**Esmer Recommendation:** Always use PATCH for updates to avoid accidental data loss. Only use PUT when the intention is to completely replace record data.

##### Delete Records

```
DELETE /v0/{baseId}/{tableIdOrName}?records[]={recordId1}&records[]={recordId2}
```

**Description:** Deletes one or more records. Maximum 10 records per request.

**Parameters:**

| Parameter | Location | Type | Required | Description |
|---|---|---|---|---|
| `records[]` | query | string[] | Yes | Array of record IDs to delete (max 10) |

**Response (200 OK):**

```json
{
  "records": [
    {
      "id": "recABC123",
      "deleted": true
    },
    {
      "id": "recDEF456",
      "deleted": true
    }
  ]
}
```

#### 5.5.2 Schema Resource (Tables, Fields, Views)

##### List Tables in a Base

```
GET /v0/meta/bases/{baseId}/tables
```

**Description:** Returns all tables in a base with their fields and views.

**Headers:** `Authorization: Bearer {token}`

**Response (200 OK):**

```json
{
  "tables": [
    {
      "id": "tblABC123",
      "name": "Tasks",
      "description": "Project task tracker",
      "primaryFieldId": "fldNAME01",
      "fields": [
        {
          "id": "fldNAME01",
          "name": "Task Name",
          "type": "singleLineText",
          "description": "Name of the task"
        },
        {
          "id": "fldSTAT02",
          "name": "Status",
          "type": "singleSelect",
          "options": {
            "choices": [
              { "id": "selNEW", "name": "New", "color": "blueLight2" },
              { "id": "selPRG", "name": "In Progress", "color": "yellowLight2" },
              { "id": "selDON", "name": "Done", "color": "greenLight2" }
            ]
          }
        },
        {
          "id": "fldDATE03",
          "name": "Due Date",
          "type": "date",
          "options": {
            "dateFormat": {
              "name": "iso",
              "format": "YYYY-MM-DD"
            }
          }
        },
        {
          "id": "fldNUM04",
          "name": "Estimated Hours",
          "type": "number",
          "options": {
            "precision": 1
          }
        },
        {
          "id": "fldFRM05",
          "name": "Days Until Due",
          "type": "formula",
          "options": {
            "expression": "DATETIME_DIFF({Due Date}, TODAY(), 'days')",
            "result": {
              "type": "number"
            }
          }
        },
        {
          "id": "fldLNK06",
          "name": "Project",
          "type": "multipleRecordLinks",
          "options": {
            "linkedTableId": "tblPROJ01",
            "isReversed": false,
            "prefersSingleRecordLink": true
          }
        },
        {
          "id": "fldTAG07",
          "name": "Tags",
          "type": "multipleSelects",
          "options": {
            "choices": [
              { "id": "selAPI", "name": "api", "color": "blueLight2" },
              { "id": "selMOB", "name": "mobile", "color": "greenLight2" },
              { "id": "selDOC", "name": "documentation", "color": "grayLight2" }
            ]
          }
        }
      ],
      "views": [
        {
          "id": "viwALL01",
          "name": "All Tasks",
          "type": "grid"
        },
        {
          "id": "viwACT02",
          "name": "Active Tasks",
          "type": "grid"
        },
        {
          "id": "viwKAN03",
          "name": "Kanban Board",
          "type": "kanban"
        }
      ]
    }
  ]
}
```

**Field Types Reference:**

| Type | Description | JSON Value Format |
|---|---|---|
| `singleLineText` | Plain text | `"string"` |
| `multilineText` | Long text with rich formatting | `"string"` |
| `number` | Numeric value | `123` or `45.67` |
| `currency` | Monetary value | `99.99` |
| `percent` | Percentage | `0.75` (for 75%) |
| `singleSelect` | Single choice from options | `"Option Name"` |
| `multipleSelects` | Multiple choices | `["Tag1", "Tag2"]` |
| `date` | Date value | `"2026-02-09"` |
| `dateTime` | Date and time | `"2026-02-09T12:00:00.000Z"` |
| `checkbox` | Boolean | `true` or `false` |
| `email` | Email address | `"user@example.com"` |
| `url` | Web URL | `"https://example.com"` |
| `phoneNumber` | Phone number | `"+1-555-0123"` |
| `multipleRecordLinks` | Links to records in another table | `["recID1", "recID2"]` |
| `multipleAttachments` | File attachments | `[{"url": "...", "filename": "..."}]` |
| `formula` | Computed field (read-only) | Varies by result type |
| `rollup` | Aggregation of linked records (read-only) | Varies by result type |
| `lookup` | Value from linked record (read-only) | Varies by source field |
| `count` | Count of linked records (read-only) | `5` |
| `autoNumber` | Auto-incrementing number (read-only) | `42` |
| `createdTime` | Record creation timestamp (read-only) | `"2026-02-09T12:00:00.000Z"` |
| `lastModifiedTime` | Last modification timestamp (read-only) | `"2026-02-09T15:30:00.000Z"` |
| `createdBy` | User who created the record (read-only) | `{"id": "usr...", "email": "...", "name": "..."}` |
| `lastModifiedBy` | User who last modified (read-only) | Same as `createdBy` |
| `rating` | Star rating (1-5 or 1-10) | `4` |
| `duration` | Duration in seconds | `3600` (for 1 hour) |
| `barcode` | Barcode data | `{"text": "123456"}` |
| `button` | Action button (read-only) | `{"label": "Open", "url": "..."}` |
| `richText` | Rich text with markdown | `"**bold** and *italic*"` |

##### Create a Table

```
POST /v0/meta/bases/{baseId}/tables
```

**Description:** Creates a new table in a base with specified fields.

**Request:**

```json
{
  "name": "Contacts",
  "description": "Customer contact database",
  "fields": [
    {
      "name": "Full Name",
      "type": "singleLineText",
      "description": "Contact's full name"
    },
    {
      "name": "Email",
      "type": "email"
    },
    {
      "name": "Company",
      "type": "singleLineText"
    },
    {
      "name": "Status",
      "type": "singleSelect",
      "options": {
        "choices": [
          { "name": "Lead", "color": "blueLight2" },
          { "name": "Active", "color": "greenLight2" },
          { "name": "Inactive", "color": "grayLight2" }
        ]
      }
    }
  ]
}
```

**Response (200 OK):**

```json
{
  "id": "tblNEW01",
  "name": "Contacts",
  "description": "Customer contact database",
  "primaryFieldId": "fldNEW01",
  "fields": [
    { "id": "fldNEW01", "name": "Full Name", "type": "singleLineText" },
    { "id": "fldNEW02", "name": "Email", "type": "email" },
    { "id": "fldNEW03", "name": "Company", "type": "singleLineText" },
    { "id": "fldNEW04", "name": "Status", "type": "singleSelect", "options": { "choices": [...] } }
  ],
  "views": [
    { "id": "viwNEW01", "name": "Grid view", "type": "grid" }
  ]
}
```

##### Create a Field

```
POST /v0/meta/bases/{baseId}/tables/{tableId}/fields
```

**Description:** Adds a new field (column) to an existing table.

**Request:**

```json
{
  "name": "Phone",
  "type": "phoneNumber",
  "description": "Primary phone number"
}
```

**Response (200 OK):**

```json
{
  "id": "fldPHN01",
  "name": "Phone",
  "type": "phoneNumber",
  "description": "Primary phone number"
}
```

##### Update a Field

```
PATCH /v0/meta/bases/{baseId}/tables/{tableId}/fields/{fieldId}
```

**Request:**

```json
{
  "name": "Phone Number",
  "description": "Updated: Primary contact phone"
}
```

##### List Views (via table schema)

Views are returned as part of the table schema response from `GET /v0/meta/bases/{baseId}/tables`. Individual view details include:

| Property | Description |
|---|---|
| `id` | View ID (starts with `viw`) |
| `name` | Display name of the view |
| `type` | `grid`, `form`, `calendar`, `gallery`, `kanban`, `timeline`, `gantt` |

To use a view's filters/sorts when listing records, pass the `view` parameter:

```
GET /v0/{baseId}/{tableIdOrName}?view=Active%20Tasks
```

#### 5.5.3 Bases Resource

##### List Bases

```
GET /v0/meta/bases
```

**Description:** Lists all bases accessible by the current token.

**Response (200 OK):**

```json
{
  "bases": [
    {
      "id": "appABC123",
      "name": "Project Tracker",
      "permissionLevel": "create"
    },
    {
      "id": "appDEF456",
      "name": "CRM",
      "permissionLevel": "edit"
    }
  ],
  "offset": null
}
```

**Permission Levels:** `none`, `read`, `comment`, `edit`, `create`, `owner`

**Error Codes (All Airtable Endpoints):**

| Code | Description |
|---|---|
| 400 | Invalid request (bad field name, invalid formula) |
| 401 | Invalid authentication (bad token) |
| 403 | Forbidden -- token lacks required scopes for the resource |
| 404 | Base, table, or record not found |
| 413 | Request body too large |
| 422 | Unprocessable -- invalid field values, type mismatch |
| 429 | Rate limited -- wait 30 seconds before retrying |
| 500 | Internal server error |
| 503 | Service unavailable |

### 5.6 Webhooks / Real-time

Airtable provides a webhooks API for real-time notifications when records change.

##### Create a Webhook

```
POST /v0/bases/{baseId}/webhooks
```

**Request:**

```json
{
  "notificationUrl": "https://api.esmer.app/webhooks/airtable",
  "specification": {
    "options": {
      "filters": {
        "dataTypes": ["tableData"],
        "recordChangeScope": "tblABC123"
      }
    }
  }
}
```

**Response (200 OK):**

```json
{
  "id": "ach00000000000001",
  "macSecretBase64": "base64-encoded-mac-secret",
  "notificationUrl": "https://api.esmer.app/webhooks/airtable",
  "expirationTime": "2026-02-16T12:00:00.000Z",
  "cursorForNextPayload": 1,
  "specification": {
    "options": {
      "filters": {
        "dataTypes": ["tableData"],
        "recordChangeScope": "tblABC123"
      }
    }
  }
}
```

**Webhook Notification Payload:**

```json
{
  "base": { "id": "appABC123" },
  "webhook": { "id": "ach00000000000001" },
  "timestamp": "2026-02-09T15:30:00.000Z"
}
```

The notification is intentionally minimal. To get the actual changes, call:

```
GET /v0/bases/{baseId}/webhooks/{webhookId}/payloads?cursor={cursor}
```

**Payload Response:**

```json
{
  "payloads": [
    {
      "timestamp": "2026-02-09T15:30:00.000Z",
      "baseTransactionNumber": 42,
      "payloadFormat": "v0",
      "actionMetadata": {
        "source": "client",
        "sourceMetadata": {
          "user": {
            "id": "usr...",
            "email": "user@example.com"
          }
        }
      },
      "changedTablesById": {
        "tblABC123": {
          "changedRecordsById": {
            "recABC123": {
              "current": {
                "cellValuesByFieldId": {
                  "fldSTAT02": { "id": "selDON", "name": "Done", "color": "greenLight2" }
                }
              },
              "previous": {
                "cellValuesByFieldId": {
                  "fldSTAT02": { "id": "selPRG", "name": "In Progress", "color": "yellowLight2" }
                }
              }
            }
          }
        }
      }
    }
  ],
  "cursor": 2,
  "mightHaveMore": false
}
```

Webhooks expire after 7 days and must be refreshed by calling:

```
POST /v0/bases/{baseId}/webhooks/{webhookId}/refresh
```

##### List Webhooks

```
GET /v0/bases/{baseId}/webhooks
```

##### Delete a Webhook

```
DELETE /v0/bases/{baseId}/webhooks/{webhookId}
```

### 5.7 Error Handling

| HTTP Code | Meaning | Esmer Action |
|---|---|---|
| 400 | Invalid request | Parse `error.message` for field name or formula issues |
| 401 | Bad token | Prompt user to re-authenticate; refresh OAuth token |
| 403 | Scope or access issue | Check that token has required scopes and base access |
| 404 | Resource not found | Verify base ID, table name, and record ID |
| 413 | Payload too large | Split batch into smaller groups |
| 422 | Invalid field values | Check field types; use `typecast: true` to auto-convert |
| 429 | Rate limited | Mandatory 30-second wait. Do not retry sooner. |
| 500/503 | Server error | Retry with backoff up to 3 times |

**Common Issue -- "Forbidden - perhaps check your credentials":** This occurs when the PAT or OAuth token lacks the required scopes for the requested operation. Verify that `data.records:read`, `data.records:write`, and `schema.bases:read` are all included. Also verify the token has access to the specific base.

### 5.8 Mobile-Specific Notes

- **Pagination:** Airtable returns a maximum of 100 records per page. For large tables, implement transparent pagination in the mobile client -- fetch pages as the user scrolls (infinite scroll pattern).
- **typecast:** Always set `typecast: true` when creating or updating records from mobile input. This allows Airtable to convert string inputs (from voice or text fields) to the correct field types (e.g., `"45.50"` becomes a number, `"2026-02-09"` becomes a date).
- **Formula Fields:** Formula, rollup, lookup, count, autoNumber, createdTime, lastModifiedTime, createdBy, and lastModifiedBy fields are read-only. Do not include them in create/update requests. Esmer should filter these out before sending write requests.
- **Record ID Discovery:** Record IDs are not displayed in the Airtable UI by default. Esmer should always use the List operation to discover record IDs before performing updates or deletes.
- **Linked Records:** When creating records with linked record fields (`multipleRecordLinks`), you must pass an array of record IDs from the linked table: `["recABC123"]`. Consider pre-fetching linked table records for user selection.
- **Offline Queue:** Queue record create/update/delete operations while offline. On reconnect, process the queue in order, respecting the 10-records-per-request batch limit and 5-req/sec rate limit (200ms spacing). Use PATCH for updates to avoid overwriting concurrent changes.
- **30-Second Rate Limit Penalty:** The 30-second mandatory wait on rate limit violation is unusually harsh. Esmer must proactively throttle requests to avoid hitting this penalty. Implement a client-side token bucket: allow 4 requests per second per base with burst capacity of 5.
- **Webhook MAC Verification:** Validate webhook payloads using the `macSecretBase64` returned when creating the webhook. Compute HMAC-SHA256 of the request body and compare it to the `X-Airtable-Content-MAC` header.

---

## Appendix A: Common Headers

### Google APIs (Docs, Sheets, Slides)

```
Authorization: Bearer {access_token}
Content-Type: application/json
Accept: application/json
```

### Microsoft Graph API (Excel)

```
Authorization: Bearer {access_token}
Content-Type: application/json
Accept: application/json
workbook-session-id: {session-id}  (optional, for session-based operations)
```

### Airtable API

```
Authorization: Bearer {personal_access_token_or_oauth_token}
Content-Type: application/json
```

## Appendix B: Esmer Offline Queue Schema

All services share a common offline queue pattern:

```json
{
  "queueId": "uuid-v4",
  "service": "google-docs|google-sheets|microsoft-excel|google-slides|airtable",
  "operation": "create|update|delete|append|batchUpdate",
  "endpoint": "/v1/documents/{id}:batchUpdate",
  "method": "POST|PUT|PATCH|DELETE|GET",
  "payload": { },
  "createdAt": "2026-02-09T12:00:00Z",
  "retryCount": 0,
  "maxRetries": 3,
  "status": "pending|processing|completed|failed",
  "dependsOn": null
}
```

**Processing Rules:**
1. Process queue items in FIFO order per service.
2. If an item fails with a retryable error (429, 500, 503), increment `retryCount` and re-queue.
3. If an item fails with a non-retryable error (400, 404, 422), mark as `failed` and notify the user.
4. For Google Docs batch updates, re-fetch the document's `revisionId` before replaying to avoid stale index references.
5. For Airtable, respect the 200ms minimum spacing between requests per base.
6. For Microsoft Excel, create a session before replaying queued operations and close it afterward.

## Appendix C: Security Considerations

1. **Token Storage:** Store all OAuth tokens (access + refresh) in platform-specific secure storage: iOS Keychain or Android Keystore. Never store tokens in UserDefaults, SharedPreferences, or local SQLite databases.
2. **PKCE:** All OAuth 2.0 flows must use PKCE (Proof Key for Code Exchange) with S256 method. This is mandatory for mobile apps where client secrets cannot be kept confidential.
3. **Scope Minimization:** Request only the scopes needed for the current operation. Use incremental authorization for Google APIs to add scopes as needed.
4. **Airtable PAT Rotation:** Recommend users rotate Personal Access Tokens every 90 days. OAuth tokens should be refreshed automatically.
5. **Webhook Validation:** Always validate webhook payloads -- Google uses channel verification, Microsoft uses `clientState`, and Airtable uses HMAC-SHA256 MAC verification.
6. **Data in Transit:** All APIs use HTTPS. Esmer must enforce TLS 1.2+ and certificate pinning for API endpoints.
7. **Cached Data:** Encrypt cached document content, spreadsheet data, and record data at rest using platform encryption (iOS Data Protection, Android EncryptedSharedPreferences).
