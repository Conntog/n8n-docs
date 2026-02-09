# Esmer (ESMO) API Integration Specification: Social Media

> **Category:** 07 - Social Media
> **Version:** 1.0.0
> **Last Updated:** 2026-02-09
> **Status:** Draft

---

## Table of Contents

1. [Category Overview](#category-overview)
2. [X / Twitter (X API v2)](#1-x--twitter-x-api-v2)
3. [LinkedIn (Community Management API)](#2-linkedin-community-management-api)
4. [Facebook Graph API (Meta Graph API)](#3-facebook-graph-api-meta-graph-api)
5. [YouTube (YouTube Data API v3)](#4-youtube-youtube-data-api-v3)
6. [Reddit (Reddit API)](#5-reddit-reddit-api)
7. [Medium (Medium API)](#6-medium-medium-api)
8. [Cross-Service Comparison Matrix](#cross-service-comparison-matrix)

---

## Category Overview

Social Media integration is a high-value capability for Esmer. As a mobile-first AI delegation assistant, Esmer enables users to manage their social media presence across platforms without switching between apps. Key use cases include:

- **Content Publishing:** Draft and post text, images, videos, and links to multiple platforms from a single interface
- **Engagement Monitoring:** Read replies, comments, mentions, and direct messages across platforms
- **Analytics Retrieval:** Pull engagement metrics (likes, shares, impressions, reach) to inform content strategy
- **Scheduling and Queuing:** Queue posts for future publication at optimal times
- **Cross-posting:** Adapt and publish content across multiple platforms with platform-specific formatting
- **Community Management:** Reply to comments, moderate discussions, and manage followers
- **Media Management:** Upload, compress, and attach photos and videos from mobile device camera rolls
- **Account Management:** Manage profiles, pages, channels, and organization accounts

Each platform has distinct API patterns, authentication models, rate limiting strategies, and content constraints. This specification documents the precise integration requirements for each.

---

## 1. X / Twitter (X API v2)

### 1.1 Service Overview

X (formerly Twitter) is the primary real-time microblogging platform. Esmer integrates with the X API v2 to enable users to compose and publish tweets, reply to conversations, search for content, like and retweet posts, send direct messages, manage lists, and retrieve user profiles. The X API has a tiered access model (Free, Basic, Pro, Enterprise) that significantly affects available endpoints and rate limits. Esmer should default to the Basic tier at minimum and recommend Pro for power users.

### 1.2 Authentication

**Method:** OAuth 2.0 with PKCE (Authorization Code Flow)

| Parameter | Value |
|---|---|
| Authorization URL | `https://twitter.com/i/oauth2/authorize` |
| Token URL | `https://api.twitter.com/2/oauth2/token` |
| Revoke URL | `https://api.twitter.com/2/oauth2/revoke` |
| Grant Type | `authorization_code` (with PKCE) |
| Client Type | Confidential Client (Web App / Automated App / Bot) |

**Required Scopes:**

| Scope | Purpose |
|---|---|
| `tweet.read` | Read tweets, timelines, and search results |
| `tweet.write` | Create, delete tweets and replies |
| `tweet.moderate.write` | Hide/unhide replies |
| `users.read` | Read user profile information |
| `follows.read` | Read following/follower lists |
| `follows.write` | Follow and unfollow users |
| `like.read` | Read liked tweets |
| `like.write` | Like and unlike tweets |
| `list.read` | Read lists and list members |
| `list.write` | Create, update, delete lists; add/remove members |
| `bookmark.read` | Read bookmarked tweets |
| `bookmark.write` | Bookmark and unbookmark tweets |
| `dm.read` | Read direct messages |
| `dm.write` | Send direct messages |
| `offline.access` | Obtain a refresh token for long-lived access |
| `space.read` | Read Spaces metadata |
| `mute.read` | Read muted users |
| `mute.write` | Mute and unmute users |
| `block.read` | Read blocked users |
| `block.write` | Block and unblock users |

**Recommendations for Esmer:**
- Always request `offline.access` to obtain refresh tokens. Access tokens expire after 2 hours.
- Request only scopes needed for the user's selected features. Start with `tweet.read`, `tweet.write`, `users.read`, and `offline.access`.
- Use PKCE with `code_challenge_method=S256`. Generate a random `code_verifier` (43-128 characters), hash with SHA-256, and Base64url-encode as `code_challenge`.
- Store the Client ID and Client Secret securely in the Esmer backend. Never embed in the mobile app binary.

**App Review:** Free tier requires no review. Basic tier ($100/month) and Pro tier ($5,000/month) provide higher limits. Enterprise tier requires sales engagement with X.

### 1.3 Base URL

```
https://api.twitter.com/2
```

Media upload endpoint (still v1.1):
```
https://upload.twitter.com/1.1/media/upload.json
```

### 1.4 Rate Limits

Rate limits vary significantly by access tier. All windows are 15-minute intervals unless otherwise noted.

**Free Tier:**

| Endpoint | Rate Limit (App) | Rate Limit (User) | Monthly Cap |
|---|---|---|---|
| POST /2/tweets | 17 req/24h per user | 17 req/24h per user | 1,500 tweets/month at app level |
| DELETE /2/tweets/:id | 50 req/15min per user | 50 req/15min per user | -- |
| GET /2/users/me | 25 req/24h per user | 25 req/24h per user | -- |

**Basic Tier ($100/month):**

| Endpoint | Rate Limit (App) | Rate Limit (User) | Monthly Cap |
|---|---|---|---|
| POST /2/tweets | 100 req/24h per user | 100 req/24h per user | 3,000 tweets/month at app level |
| DELETE /2/tweets/:id | 50 req/15min per user | 50 req/15min per user | -- |
| GET /2/tweets/:id | 300 req/15min per app | 900 req/15min per user | -- |
| GET /2/tweets/search/recent | 60 req/15min per app | 60 req/15min per user | 10,000 tweets/month read |
| GET /2/users/:id | 300 req/15min per app | 900 req/15min per user | -- |
| GET /2/users/me | 25 req/15min per user | 25 req/15min per user | -- |
| GET /2/users/by/username/:username | 300 req/15min per app | 900 req/15min per user | -- |
| POST /2/dm_conversations | 200 req/15min per app | 200 req/15min per user | -- |

**Pro Tier ($5,000/month):**

| Endpoint | Rate Limit (App) | Rate Limit (User) | Monthly Cap |
|---|---|---|---|
| POST /2/tweets | 100 req/24h per user | 100 req/24h per user | 300,000 tweets/month at app level |
| GET /2/tweets/search/recent | 300 req/15min per app | 300 req/15min per user | 1,000,000 tweets/month read |
| GET /2/tweets/search/all (full archive) | 300 req/15min per app | 1 req/1sec per user | 1,000,000 tweets/month read |
| GET /2/tweets/counts/recent | 300 req/15min per app | 300 req/15min per user | -- |
| GET /2/tweets/counts/all | 300 req/15min per app | 300 req/15min per user | -- |
| All GET endpoints | 10x Basic tier | 10x Basic tier | -- |

**Rate Limit Headers:**
- `x-rate-limit-limit` -- Maximum requests allowed in the window
- `x-rate-limit-remaining` -- Requests remaining in the current window
- `x-rate-limit-reset` -- UTC epoch timestamp when the window resets

**Retry Strategy:** On `429 Too Many Requests`, read the `x-rate-limit-reset` header and wait until that timestamp. Do not use exponential backoff -- the reset time is deterministic.

### 1.5 API Endpoints

#### Tweets

**Create a Tweet**
```
POST /2/tweets
```
- **Description:** Create a new tweet, optionally as a reply, quote tweet, or with media/poll.
- **Tier Access:** Free, Basic, Pro, Enterprise
- **Request Body:**
```json
{
  "text": "Hello from Esmer!",
  "reply": {
    "in_reply_to_tweet_id": "1234567890"
  },
  "quote_tweet_id": "9876543210",
  "media": {
    "media_ids": ["1455952740635586573"],
    "tagged_user_ids": ["2244994945"]
  },
  "poll": {
    "options": ["Option A", "Option B", "Option C"],
    "duration_minutes": 1440
  },
  "reply_settings": "mentionedUsers",
  "geo": {
    "place_id": "5a110d312052166f"
  }
}
```
- **Response (201 Created):**
```json
{
  "data": {
    "id": "1445880548472328192",
    "text": "Hello from Esmer!",
    "edit_history_tweet_ids": ["1445880548472328192"]
  }
}
```
- **Constraints:**
  - `text` max length: 280 characters (Free/Basic/Pro), 25,000 characters (Premium/X Premium subscribers)
  - `poll.options`: 2-4 options, each 1-25 characters
  - `poll.duration_minutes`: 5-10080 (7 days)
  - `media.media_ids`: max 4 images, or 1 GIF, or 1 video
- **Error Codes:**
  - `400` -- Invalid request (malformed JSON, exceeded character limit)
  - `401` -- Unauthorized (invalid or expired token)
  - `403` -- Forbidden (scope missing, account suspended, duplicate tweet)
  - `429` -- Rate limit exceeded

**Delete a Tweet**
```
DELETE /2/tweets/{id}
```
- **Description:** Delete a tweet owned by the authenticated user.
- **Tier Access:** Free, Basic, Pro, Enterprise
- **Path Parameters:**
  - `id` (required) -- The tweet ID to delete
- **Response (200 OK):**
```json
{
  "data": {
    "deleted": true
  }
}
```

**Get a Tweet**
```
GET /2/tweets/{id}
```
- **Description:** Retrieve a single tweet by ID with optional expansions.
- **Tier Access:** Basic, Pro, Enterprise (not available on Free)
- **Path Parameters:**
  - `id` (required) -- The tweet ID
- **Query Parameters:**
  - `tweet.fields` -- Comma-separated: `attachments,author_id,context_annotations,conversation_id,created_at,entities,geo,id,in_reply_to_user_id,lang,possibly_sensitive,public_metrics,referenced_tweets,reply_settings,source,text,withheld`
  - `expansions` -- `attachments.poll_ids,attachments.media_keys,author_id,entities.mentions.username,geo.place_id,in_reply_to_user_id,referenced_tweets.id,referenced_tweets.id.author_id`
  - `media.fields` -- `duration_ms,height,media_key,preview_image_url,type,url,width,alt_text,public_metrics`
  - `user.fields` -- `created_at,description,entities,id,location,name,pinned_tweet_id,profile_image_url,protected,public_metrics,url,username,verified`
  - `poll.fields` -- `duration_minutes,end_datetime,id,options,voting_status`
  - `place.fields` -- `contained_within,country,country_code,full_name,geo,id,name,place_type`
- **Response (200 OK):**
```json
{
  "data": {
    "id": "1445880548472328192",
    "text": "Hello from Esmer!",
    "author_id": "2244994945",
    "created_at": "2026-01-15T12:00:00.000Z",
    "public_metrics": {
      "retweet_count": 5,
      "reply_count": 2,
      "like_count": 42,
      "quote_count": 1,
      "bookmark_count": 3,
      "impression_count": 1500
    },
    "conversation_id": "1445880548472328192",
    "lang": "en",
    "source": "Esmer",
    "reply_settings": "everyone"
  },
  "includes": {
    "users": [
      {
        "id": "2244994945",
        "name": "Esmer User",
        "username": "esmeruser",
        "profile_image_url": "https://pbs.twimg.com/profile_images/..."
      }
    ]
  }
}
```

**Get Multiple Tweets**
```
GET /2/tweets
```
- **Description:** Retrieve multiple tweets by IDs (up to 100).
- **Tier Access:** Basic, Pro, Enterprise
- **Query Parameters:**
  - `ids` (required) -- Comma-separated tweet IDs (max 100)
  - Same `tweet.fields`, `expansions`, `media.fields`, `user.fields`, `poll.fields`, `place.fields` as single tweet lookup
- **Response:** Same structure with `data` as an array.

**Search Recent Tweets**
```
GET /2/tweets/search/recent
```
- **Description:** Search tweets from the past 7 days.
- **Tier Access:** Basic, Pro, Enterprise
- **Query Parameters:**
  - `query` (required) -- Search query (max 512 characters on Basic, 1024 on Pro). Supports operators: `from:`, `to:`, `is:retweet`, `is:reply`, `has:media`, `has:images`, `has:videos`, `has:links`, `lang:`, `place:`, `url:`, `-is:retweet`, `#hashtag`, `@mention`, `"exact phrase"`
  - `start_time` -- ISO 8601 timestamp (oldest allowed, within 7 days)
  - `end_time` -- ISO 8601 timestamp
  - `since_id` -- Returns results with tweet ID greater than this
  - `until_id` -- Returns results with tweet ID less than this
  - `max_results` -- 10-100 (default 10)
  - `next_token` -- Pagination token
  - `sort_order` -- `recency` or `relevancy`
  - Same `tweet.fields`, `expansions`, etc.
- **Response (200 OK):**
```json
{
  "data": [
    {
      "id": "1445880548472328192",
      "text": "Esmer is amazing!",
      "author_id": "2244994945",
      "created_at": "2026-02-08T10:30:00.000Z"
    }
  ],
  "meta": {
    "newest_id": "1445880548472328192",
    "oldest_id": "1445880548472328192",
    "result_count": 1,
    "next_token": "b26v89c19zqg8o3fpwbkuvnag9r7bsi2grfm6kxb1ai14"
  }
}
```

**Search Full Archive Tweets**
```
GET /2/tweets/search/all
```
- **Description:** Search the complete archive of public tweets (all time).
- **Tier Access:** Pro and Enterprise only
- **Query Parameters:** Same as recent search, plus `start_time` can go back to 2006.
- **Response:** Same structure as recent search.

**Tweet Counts (Recent)**
```
GET /2/tweets/counts/recent
```
- **Description:** Get count of tweets matching a query in the last 7 days, bucketed by time.
- **Tier Access:** Pro, Enterprise
- **Query Parameters:**
  - `query` (required)
  - `start_time`, `end_time`
  - `granularity` -- `minute`, `hour`, `day` (default `hour`)
- **Response (200 OK):**
```json
{
  "data": [
    { "start": "2026-02-08T00:00:00.000Z", "end": "2026-02-08T01:00:00.000Z", "tweet_count": 42 }
  ],
  "meta": { "total_tweet_count": 500 }
}
```

**Hide a Reply**
```
PUT /2/tweets/{id}/hidden
```
- **Description:** Hide or unhide a reply to the authenticated user's tweet.
- **Tier Access:** Basic, Pro, Enterprise
- **Request Body:**
```json
{ "hidden": true }
```
- **Response (200 OK):**
```json
{ "data": { "hidden": true } }
```

#### Likes

**Like a Tweet**
```
POST /2/users/{id}/likes
```
- **Description:** Like a tweet on behalf of the authenticated user.
- **Tier Access:** Basic, Pro, Enterprise
- **Path Parameters:**
  - `id` -- The authenticated user's ID
- **Request Body:**
```json
{ "tweet_id": "1445880548472328192" }
```
- **Response (200 OK):**
```json
{ "data": { "liked": true } }
```
- **Rate Limit:** 200 requests per 15 minutes per user; 1,000 likes per 24 hours per user

**Unlike a Tweet**
```
DELETE /2/users/{id}/likes/{tweet_id}
```
- **Response (200 OK):**
```json
{ "data": { "liked": false } }
```

**Get Liked Tweets**
```
GET /2/users/{id}/liked_tweets
```
- **Description:** Retrieve tweets liked by a user.
- **Tier Access:** Basic, Pro, Enterprise
- **Query Parameters:**
  - `max_results` -- 10-100
  - `pagination_token`
  - Standard `tweet.fields`, `expansions`
- **Rate Limit:** 75 requests per 15 minutes per user

#### Retweets

**Retweet**
```
POST /2/users/{id}/retweets
```
- **Request Body:**
```json
{ "tweet_id": "1445880548472328192" }
```
- **Response (200 OK):**
```json
{ "data": { "retweeted": true } }
```
- **Rate Limit:** 300 requests per 15 minutes per user; 1,000 retweets per 24 hours per user

**Undo Retweet**
```
DELETE /2/users/{id}/retweets/{source_tweet_id}
```
- **Response (200 OK):**
```json
{ "data": { "retweeted": false } }
```

#### Bookmarks

**Bookmark a Tweet**
```
POST /2/users/{id}/bookmarks
```
- **Request Body:**
```json
{ "tweet_id": "1445880548472328192" }
```
- **Response (200 OK):**
```json
{ "data": { "bookmarked": true } }
```

**Remove Bookmark**
```
DELETE /2/users/{id}/bookmarks/{tweet_id}
```
- **Response (200 OK):**
```json
{ "data": { "bookmarked": false } }
```

**Get Bookmarks**
```
GET /2/users/{id}/bookmarks
```
- **Query Parameters:** `max_results` (1-100), `pagination_token`, standard `tweet.fields`, `expansions`
- **Rate Limit:** 180 requests per 15 minutes per user

#### Users

**Get User by ID**
```
GET /2/users/{id}
```
- **Description:** Retrieve a user's profile information.
- **Tier Access:** Basic, Pro, Enterprise
- **Query Parameters:**
  - `user.fields` -- `created_at,description,entities,id,location,most_recent_tweet_id,name,pinned_tweet_id,profile_image_url,protected,public_metrics,url,username,verified,verified_type,withheld`
  - `expansions` -- `pinned_tweet_id`
  - `tweet.fields` (for expanded pinned tweet)
- **Response (200 OK):**
```json
{
  "data": {
    "id": "2244994945",
    "name": "Esmer User",
    "username": "esmeruser",
    "created_at": "2020-01-01T00:00:00.000Z",
    "description": "Using Esmer to manage my social media",
    "profile_image_url": "https://pbs.twimg.com/profile_images/..._normal.jpg",
    "public_metrics": {
      "followers_count": 5000,
      "following_count": 300,
      "tweet_count": 12000,
      "listed_count": 50,
      "like_count": 8000
    },
    "verified": false,
    "verified_type": "none",
    "location": "San Francisco, CA",
    "url": "https://esmer.app"
  }
}
```

**Get User by Username**
```
GET /2/users/by/username/{username}
```
- Same query parameters and response structure as Get User by ID.

**Get Authenticated User**
```
GET /2/users/me
```
- **Description:** Retrieve the authenticated user's own profile.
- **Tier Access:** Free, Basic, Pro, Enterprise
- Same query parameters and response structure.
- **Rate Limit:** 25 requests per 24 hours (Free), 75 per 15 minutes (Basic+)

**Get Multiple Users**
```
GET /2/users
```
- **Query Parameters:** `ids` (required, comma-separated, max 100), same `user.fields` and `expansions`

**Get Multiple Users by Username**
```
GET /2/users/by
```
- **Query Parameters:** `usernames` (required, comma-separated, max 100)

#### Follows

**Follow a User**
```
POST /2/users/{id}/following
```
- **Request Body:**
```json
{ "target_user_id": "2244994945" }
```
- **Response (200 OK):**
```json
{ "data": { "following": true, "pending_follow": false } }
```
- **Rate Limit:** 15 requests per 15 minutes per user; 400 follows per 24 hours per user

**Unfollow a User**
```
DELETE /2/users/{source_user_id}/following/{target_user_id}
```
- **Response (200 OK):**
```json
{ "data": { "following": false } }
```

**Get Followers**
```
GET /2/users/{id}/followers
```
- **Query Parameters:** `max_results` (1-1000), `pagination_token`, standard `user.fields`
- **Rate Limit:** 15 requests per 15 minutes per user

**Get Following**
```
GET /2/users/{id}/following
```
- Same parameters and rate limit as Get Followers.

#### Direct Messages

**Create a DM Conversation and Send Message**
```
POST /2/dm_conversations
```
- **Description:** Create a new one-on-one or group DM conversation with an initial message.
- **Tier Access:** Basic, Pro, Enterprise
- **Request Body (one-on-one):**
```json
{
  "conversation_type": "Group",
  "participant_ids": ["2244994945", "6253282"],
  "message": {
    "text": "Hello from Esmer!",
    "attachments": [
      { "media_id": "1455952740635586573" }
    ]
  }
}
```
- **Response (201 Created):**
```json
{
  "data": {
    "dm_conversation_id": "1580705810800955392",
    "dm_event_id": "1580705810800955393"
  }
}
```
- **Rate Limit:** 200 requests per 15 minutes per user; 1,000 DMs per 24 hours per user

**Send Message to Existing Conversation**
```
POST /2/dm_conversations/{dm_conversation_id}/messages
```
- **Request Body:**
```json
{
  "text": "Follow-up message",
  "attachments": [
    { "media_id": "1455952740635586573" }
  ]
}
```
- **Response (201 Created):**
```json
{
  "data": {
    "dm_conversation_id": "1580705810800955392",
    "dm_event_id": "1580705810800955394"
  }
}
```

**Send DM to a User Directly**
```
POST /2/dm_conversations/with/{participant_id}/messages
```
- Same request body and response as above.
- Creates a one-on-one conversation if none exists.

**Get DM Events**
```
GET /2/dm_events
```
- **Description:** Retrieve DM events for the authenticated user.
- **Query Parameters:**
  - `dm_event.fields` -- `id,text,event_type,dm_conversation_id,created_at,sender_id,attachments,participant_ids,referenced_tweets`
  - `event_types` -- `MessageCreate,ParticipantsJoin,ParticipantsLeave`
  - `max_results` -- 1-100 (default 100)
  - `pagination_token`
  - `expansions` -- `attachments.media_keys,participant_ids,referenced_tweets.id,sender_id`
- **Rate Limit:** 300 requests per 15 minutes per user

**Get DM Events for a Conversation**
```
GET /2/dm_conversations/{id}/dm_events
```
- Same parameters as above, scoped to a specific conversation.

#### Lists

**Create a List**
```
POST /2/lists
```
- **Request Body:**
```json
{
  "name": "Esmer Favorites",
  "description": "Curated by Esmer AI",
  "private": false
}
```
- **Response (200 OK):**
```json
{ "data": { "id": "1441162269824405510", "name": "Esmer Favorites" } }
```

**Delete a List**
```
DELETE /2/lists/{id}
```
- **Response (200 OK):**
```json
{ "data": { "deleted": true } }
```

**Update a List**
```
PUT /2/lists/{id}
```
- **Request Body:**
```json
{ "name": "Updated Name", "description": "Updated description", "private": true }
```

**Add Member to a List**
```
POST /2/lists/{id}/members
```
- **Request Body:**
```json
{ "user_id": "2244994945" }
```
- **Response (200 OK):**
```json
{ "data": { "is_member": true } }
```

**Remove Member from a List**
```
DELETE /2/lists/{id}/members/{user_id}
```
- **Response (200 OK):**
```json
{ "data": { "is_member": false } }
```

**Get List Members**
```
GET /2/lists/{id}/members
```
- **Query Parameters:** `max_results` (1-100), `pagination_token`, standard `user.fields`

**Get Lists Owned by User**
```
GET /2/users/{id}/owned_lists
```
- **Query Parameters:** `max_results` (1-100), `pagination_token`, `list.fields` (`created_at,description,follower_count,id,member_count,name,owner_id,private`)

**Get List Tweets**
```
GET /2/lists/{id}/tweets
```
- **Query Parameters:** `max_results` (1-100), `pagination_token`, standard `tweet.fields`, `expansions`
- **Rate Limit:** 900 requests per 15 minutes per user

#### Mutes and Blocks

**Mute a User**
```
POST /2/users/{id}/muting
```
- **Request Body:**
```json
{ "target_user_id": "2244994945" }
```

**Unmute a User**
```
DELETE /2/users/{source_user_id}/muting/{target_user_id}
```

**Get Muted Users**
```
GET /2/users/{id}/muting
```

**Block a User**
```
POST /2/users/{id}/blocking
```
- **Request Body:**
```json
{ "target_user_id": "2244994945" }
```

**Unblock a User**
```
DELETE /2/users/{source_user_id}/blocking/{target_user_id}
```

**Get Blocked Users**
```
GET /2/users/{id}/blocking
```

#### Media Upload (v1.1)

Media upload uses the v1.1 endpoint since v2 does not yet have a media upload endpoint.

**Simple Media Upload (images < 5 MB, GIFs < 15 MB)**
```
POST https://upload.twitter.com/1.1/media/upload.json
```
- **Content-Type:** `multipart/form-data`
- **Form Parameters:**
  - `media_data` -- Base64-encoded file data
  - `media_category` -- `tweet_image`, `tweet_gif`, `tweet_video`, `dm_image`, `dm_gif`, `dm_video`
- **Response (200 OK):**
```json
{
  "media_id": 1455952740635586573,
  "media_id_string": "1455952740635586573",
  "size": 35840,
  "expires_after_secs": 86400,
  "image": {
    "image_type": "image/jpeg",
    "w": 1200,
    "h": 675
  }
}
```

**Chunked Media Upload (videos, large GIFs)**

Step 1 -- INIT:
```
POST https://upload.twitter.com/1.1/media/upload.json
```
- **Form Parameters:**
  - `command=INIT`
  - `total_bytes` -- Total file size in bytes
  - `media_type` -- MIME type (e.g., `video/mp4`)
  - `media_category` -- `tweet_video`
- **Response:**
```json
{ "media_id": 1455952740635586573, "media_id_string": "1455952740635586573", "expires_after_secs": 86400 }
```

Step 2 -- APPEND (repeat for each chunk):
```
POST https://upload.twitter.com/1.1/media/upload.json
```
- **Form Parameters:**
  - `command=APPEND`
  - `media_id` -- From INIT response
  - `segment_index` -- 0-based chunk index
  - `media_data` -- Base64-encoded chunk (max 5 MB per chunk recommended)
- **Response:** `204 No Content` on success

Step 3 -- FINALIZE:
```
POST https://upload.twitter.com/1.1/media/upload.json
```
- **Form Parameters:**
  - `command=FINALIZE`
  - `media_id` -- From INIT response
- **Response:**
```json
{
  "media_id": 1455952740635586573,
  "media_id_string": "1455952740635586573",
  "processing_info": {
    "state": "pending",
    "check_after_secs": 5
  }
}
```

Step 4 -- STATUS (poll until processing completes):
```
GET https://upload.twitter.com/1.1/media/upload.json?command=STATUS&media_id={media_id}
```
- **Response (processing):**
```json
{
  "media_id": 1455952740635586573,
  "processing_info": {
    "state": "in_progress",
    "check_after_secs": 10,
    "progress_percent": 45
  }
}
```
- **Response (succeeded):**
```json
{
  "media_id": 1455952740635586573,
  "processing_info": {
    "state": "succeeded"
  }
}
```

**Media Constraints:**

| Type | Max Size | Max Duration | Formats |
|---|---|---|---|
| Image | 5 MB | -- | JPEG, PNG, GIF, WEBP |
| Animated GIF | 15 MB | -- | GIF |
| Video | 512 MB | 140 seconds | MP4 (H.264 video, AAC audio) |
| Video (ads) | 1 GB | 140 seconds | MP4 |

**Add Alt Text to Media**
```
POST https://upload.twitter.com/1.1/media/metadata/create.json
```
- **Content-Type:** `application/json`
- **Request Body:**
```json
{
  "media_id": "1455952740635586573",
  "alt_text": {
    "text": "A photo of the Esmer app dashboard showing task delegation"
  }
}
```
- Alt text max length: 1,000 characters

#### Timelines

**Get User Tweets (Timeline)**
```
GET /2/users/{id}/tweets
```
- **Description:** Retrieve tweets authored by a specific user.
- **Query Parameters:**
  - `max_results` -- 5-100
  - `pagination_token`
  - `start_time`, `end_time` -- ISO 8601
  - `since_id`, `until_id`
  - `exclude` -- `retweets`, `replies` (comma-separated)
  - Standard `tweet.fields`, `expansions`, etc.
- **Rate Limit:** 900 requests per 15 minutes per user; 1,500 per 15 minutes per app

**Get User Mentions**
```
GET /2/users/{id}/mentions
```
- **Description:** Retrieve tweets mentioning a specific user.
- Same parameters as user tweets timeline.
- **Rate Limit:** 450 requests per 15 minutes per user

**Get Reverse Chronological Timeline**
```
GET /2/users/{id}/reverse_chronological_timeline
```
- **Description:** Retrieve the authenticated user's home timeline in reverse chronological order (tweets from people they follow).
- **Tier Access:** Basic, Pro, Enterprise
- Same parameters.
- **Rate Limit:** 180 requests per 15 minutes per user

### 1.6 Webhooks / Real-time

**Account Activity API (AAAPI):**

The Account Activity API provides real-time webhooks for events on the authenticated user's account. It is available at Enterprise tier only.

**Register a Webhook URL**
```
POST /1.1/account_activity/all/{env_name}/webhooks.json?url={encoded_url}
```
- X will send a CRC challenge (`GET` with `crc_token`) to verify the URL.
- The webhook must respond with: `{ "response_token": "sha256=BASE64_HMAC_SHA256_OF_CRC_TOKEN" }`

**Subscribe to User Events**
```
POST /1.1/account_activity/all/{env_name}/subscriptions.json
```

**Webhook Events Delivered:**
- `tweet_create_events` -- New tweets by or mentioning the user
- `favorite_events` -- Likes on the user's tweets
- `follow_events` -- New followers and unfollows
- `direct_message_events` -- New DMs
- `direct_message_indicate_typing_events` -- Typing indicators
- `direct_message_mark_read_events` -- Read receipts
- `tweet_delete_events` -- Deleted tweets
- `mute_events`, `block_events` -- Mute/block changes

**Filtered Stream (alternative for Pro/Enterprise):**
```
GET /2/tweets/search/stream
```
- **Description:** Real-time streaming endpoint delivering tweets matching your filter rules.
- **Rules Management:**
  - `POST /2/tweets/search/stream/rules` -- Add rules
  - `GET /2/tweets/search/stream/rules` -- List rules
  - Request body for adding: `{ "add": [{ "value": "from:esmeruser OR @esmeruser", "tag": "esmer-mentions" }] }`
- **Pro tier:** 5 rules, 25 rules for Enterprise
- Returns newline-delimited JSON stream.

**Esmer Recommendation:** For Basic tier, poll the mentions and timeline endpoints every 60 seconds. For Pro tier, use the Filtered Stream for near-real-time notifications. Store last `since_id` to avoid processing duplicates.

### 1.7 Error Handling

| HTTP Code | Error Type | Description | Retry? |
|---|---|---|---|
| 400 | `InvalidRequest` | Malformed request, invalid parameters | No -- fix request |
| 401 | `Unauthorized` | Token expired or invalid | Yes -- refresh token, then retry |
| 403 | `Forbidden` | Missing scope, suspended account, duplicate content | No -- check scope/content |
| 404 | `NotFound` | Tweet/user deleted or does not exist | No |
| 409 | `Conflict` | Already liked, already retweeted | No -- idempotent, treat as success |
| 429 | `TooManyRequests` | Rate limit exceeded | Yes -- wait until `x-rate-limit-reset` |
| 500 | `InternalError` | X server error | Yes -- exponential backoff |
| 502 | `BadGateway` | X server overloaded | Yes -- exponential backoff |
| 503 | `ServiceUnavailable` | X temporarily down | Yes -- exponential backoff |

**Error Response Format:**
```json
{
  "errors": [
    {
      "message": "You are not permitted to perform this action.",
      "type": "https://api.twitter.com/2/problems/not-authorized-for-resource",
      "title": "Not Authorized",
      "detail": "Sorry, you are not authorized to see the Tweet with id: [1234567890].",
      "status": 403
    }
  ]
}
```

### 1.8 Mobile-Specific Notes

- **Media Upload from Device:** Use the chunked upload flow for all video uploads from mobile. For images, use the simple upload with Base64 encoding. Compress images to JPEG quality 80 and resize to max 1200px width before upload to reduce data usage.
- **Video Compression:** Transcode video to H.264 MP4 at 720p (1280x720), 30fps, 2 Mbps bitrate before upload. Use React Native libraries like `react-native-video-processing` or `react-native-compressor`.
- **Content Preview:** Before posting, render a preview card showing the tweet text with character count (280 limit indicator), attached media thumbnails, and mentioned users. Use the `users/by` endpoint to validate @mentions and show profile images.
- **Offline Drafts:** Queue tweets locally when offline. On reconnection, check for duplicate content (X rejects exact duplicate tweets within a short window) and post sequentially with 1-second delays.
- **Thread Composition:** For multi-tweet threads, post sequentially using the `reply.in_reply_to_tweet_id` field, using the ID from each previous tweet response. Show thread progress indicator in the UI.
- **Character Counting:** URLs always count as 23 characters regardless of length (t.co wrapping). Use Twitter's text parsing library rules: each emoji counts as 2 characters, CJK characters count as 2.
- **Poll Creation:** Provide a mobile-friendly poll builder with 2-4 option fields and a duration picker (5 minutes to 7 days).

---

## 2. LinkedIn (Community Management API)

### 2.1 Service Overview

LinkedIn is the dominant professional networking platform. Esmer integrates with LinkedIn's Community Management API (versioned, currently `202404`) to enable users to publish posts as individuals or organizations, share articles and media, manage company page content, read post analytics, and interact with comments and reactions. The LinkedIn API is accessed through Microsoft's developer portal and requires specific product approvals (Share on LinkedIn, Sign In with LinkedIn, Advertising API) depending on functionality needed.

### 2.2 Authentication

**Method:** OAuth 2.0 (Authorization Code Flow)

| Parameter | Value |
|---|---|
| Authorization URL | `https://www.linkedin.com/oauth/v2/authorization` |
| Token URL | `https://www.linkedin.com/oauth/v2/accessToken` |
| Revoke URL | Not supported (tokens expire naturally) |
| Grant Type | `authorization_code` |
| Token Lifetime | 60 days (access token), 365 days (refresh token with approved apps) |

**Required Scopes (Community Management OAuth2):**

| Scope | Purpose | Requires App Review? |
|---|---|---|
| `openid` | OpenID Connect sign-in | No |
| `profile` | Read basic profile (name, photo, headline) | No |
| `email` | Read primary email address | No |
| `w_member_social` | Create, update, delete posts as the authenticated member | No |
| `r_organization_social` | Read posts and analytics for managed organizations | Yes |
| `w_organization_social` | Create, update, delete posts as a managed organization | Yes |
| `rw_organization_admin` | Manage organization pages (admins only) | Yes |
| `r_ads` | Read advertising account data | Yes |
| `r_ads_reporting` | Read advertising analytics | Yes |
| `r_1st_connections_size` | Read first-degree connection count | No |
| `r_basicprofile` | Legacy scope for basic profile | No (legacy) |
| `r_liteprofile` | Legacy scope for lite profile | No (legacy) |

**Recommendations for Esmer:**
- Use the Community Management OAuth2 method for new integrations.
- Start with `openid`, `profile`, `email`, and `w_member_social` for individual users.
- For organization posting, apply for the Community Management App Review through LinkedIn's developer portal. This requires demonstrating a legitimate use case and may take 2-4 weeks.
- Access tokens last 60 days. Implement proactive refresh 7 days before expiry.
- LinkedIn does not provide a token revocation endpoint; tokens expire naturally or can be invalidated by the user from their LinkedIn settings.

### 2.3 Base URL

```
https://api.linkedin.com/rest
```

Legacy (v2) base URL (still used for some endpoints):
```
https://api.linkedin.com/v2
```

**Required Headers (all requests):**
```
Authorization: Bearer {access_token}
LinkedIn-Version: 202404
X-Restli-Protocol-Version: 2.0.0
Content-Type: application/json
```

### 2.4 Rate Limits

LinkedIn uses a daily rate limit system rather than per-minute windows.

| Category | Limit | Window |
|---|---|---|
| Application-level calls | 100,000 requests/day | 24-hour rolling |
| Member-level rate limit | Varies by endpoint | Per endpoint |
| POST create share (member) | 150 posts/day per member | 24-hour rolling |
| POST create share (organization) | 150 posts/day per org | 24-hour rolling |
| GET profile | 100 requests/day per member token | 24-hour rolling |
| GET organization lookup | 1,000 requests/day per app | 24-hour rolling |
| Image upload | 100 uploads/day per member | 24-hour rolling |
| Video upload | 50 uploads/day per member | 24-hour rolling |
| Comment create | 50 comments/day per member | 24-hour rolling |

**Rate Limit Headers:**
- `X-RateLimit-Limit` -- Maximum requests
- `X-RateLimit-Remaining` -- Remaining requests
- `X-RateLimit-Reset` -- Seconds until reset

**Throttle Response:** `429 Too Many Requests` with a `Retry-After` header (seconds).

### 2.5 API Endpoints

#### Posts

**Create a Post (Text)**
```
POST /rest/posts
```
- **Description:** Create a new post as a person or organization.
- **Request Body (person post with text):**
```json
{
  "author": "urn:li:person:{personId}",
  "commentary": "Excited to share our latest update! #esmer #ai",
  "visibility": "PUBLIC",
  "distribution": {
    "feedDistribution": "MAIN_FEED",
    "targetEntities": [],
    "thirdPartyDistributionChannels": []
  },
  "lifecycleState": "PUBLISHED",
  "isReshareDisabledByAuthor": false
}
```
- **Request Body (organization post):**
```json
{
  "author": "urn:li:organization:{organizationId}",
  "commentary": "Company update from Esmer",
  "visibility": "PUBLIC",
  "distribution": {
    "feedDistribution": "MAIN_FEED"
  },
  "lifecycleState": "PUBLISHED"
}
```
- **Response (201 Created):**
  - Header `x-restli-id` contains the post URN: `urn:li:share:{shareId}`
  - Body may be empty or contain the post URN.
- **Constraints:**
  - `commentary` max length: 3,000 characters
  - `visibility`: `PUBLIC`, `CONNECTIONS` (person only), `LOGGED_IN` (organization only)
- **Error Codes:**
  - `401` -- Invalid or expired token
  - `403` -- Missing `w_member_social` or `w_organization_social` scope
  - `422` -- Invalid URN, malformed request

**Create a Post with Article/Link**
```
POST /rest/posts
```
- **Request Body:**
```json
{
  "author": "urn:li:person:{personId}",
  "commentary": "Great article on AI delegation",
  "visibility": "PUBLIC",
  "distribution": {
    "feedDistribution": "MAIN_FEED"
  },
  "content": {
    "article": {
      "source": "https://esmer.app/blog/ai-delegation",
      "title": "AI Delegation with Esmer",
      "description": "Learn how Esmer delegates tasks to cloud APIs"
    }
  },
  "lifecycleState": "PUBLISHED"
}
```

**Create a Post with Image**

Step 1 -- Initialize image upload:
```
POST /rest/images?action=initializeUpload
```
- **Request Body:**
```json
{
  "initializeUploadRequest": {
    "owner": "urn:li:person:{personId}"
  }
}
```
- **Response (200 OK):**
```json
{
  "value": {
    "uploadUrlExpiresAt": 1700000000000,
    "uploadUrl": "https://www.linkedin.com/dms-uploads/...",
    "image": "urn:li:image:{imageId}"
  }
}
```

Step 2 -- Upload binary image to the provided URL:
```
PUT {uploadUrl}
```
- **Headers:**
  - `Content-Type: application/octet-stream`
  - `Authorization: Bearer {access_token}`
- **Body:** Raw binary image data
- **Response:** `201 Created`
- **Constraints:** Max image size 10 MB. Formats: JPEG, PNG, GIF (static).

Step 3 -- Create the post referencing the image:
```
POST /rest/posts
```
- **Request Body:**
```json
{
  "author": "urn:li:person:{personId}",
  "commentary": "Check out this image",
  "visibility": "PUBLIC",
  "distribution": {
    "feedDistribution": "MAIN_FEED"
  },
  "content": {
    "media": {
      "title": "Image Title",
      "id": "urn:li:image:{imageId}"
    }
  },
  "lifecycleState": "PUBLISHED"
}
```

**Create a Post with Video**

Step 1 -- Initialize video upload:
```
POST /rest/videos?action=initializeUpload
```
- **Request Body:**
```json
{
  "initializeUploadRequest": {
    "owner": "urn:li:person:{personId}",
    "fileSizeBytes": 52428800,
    "uploadCaptions": false,
    "uploadThumbnail": false
  }
}
```
- **Response (200 OK):**
```json
{
  "value": {
    "uploadInstructions": [
      {
        "uploadUrl": "https://www.linkedin.com/dms-uploads/...",
        "firstByte": 0,
        "lastByte": 4194303
      },
      {
        "uploadUrl": "https://www.linkedin.com/dms-uploads/...",
        "firstByte": 4194304,
        "lastByte": 8388607
      }
    ],
    "video": "urn:li:video:{videoId}",
    "uploadToken": "..."
  }
}
```

Step 2 -- Upload each chunk using the provided upload URLs:
```
PUT {uploadUrl}
```
- **Headers:** `Content-Type: application/octet-stream`
- **Body:** Binary chunk matching the byte range
- **Response:** `201 Created` with `ETag` header

Step 3 -- Finalize the upload:
```
POST /rest/videos?action=finalizeUpload
```
- **Request Body:**
```json
{
  "finalizeUploadRequest": {
    "video": "urn:li:video:{videoId}",
    "uploadToken": "...",
    "uploadedPartIds": ["etag1", "etag2"]
  }
}
```

Step 4 -- Create the post referencing the video (same as image, using `urn:li:video:{videoId}`).

- **Constraints:** Max video size 200 MB (API), max duration 15 minutes, formats: MP4 (H.264 + AAC).

**Delete a Post**
```
DELETE /rest/posts/{postUrn}
```
- **Path Parameters:**
  - `postUrn` -- URL-encoded post URN (e.g., `urn%3Ali%3Ashare%3A{shareId}`)
- **Response:** `204 No Content`

**Get a Post**
```
GET /rest/posts/{postUrn}
```
- **Response (200 OK):**
```json
{
  "author": "urn:li:person:{personId}",
  "commentary": "Post text content",
  "visibility": "PUBLIC",
  "lifecycleState": "PUBLISHED",
  "createdAt": 1700000000000,
  "lastModifiedAt": 1700000000000,
  "id": "urn:li:share:{shareId}",
  "distribution": {
    "feedDistribution": "MAIN_FEED"
  },
  "content": {},
  "isReshareDisabledByAuthor": false
}
```

#### Comments

**Create a Comment on a Post**
```
POST /rest/socialActions/{postUrn}/comments
```
- **Request Body:**
```json
{
  "actor": "urn:li:person:{personId}",
  "message": {
    "text": "Great post! Thanks for sharing."
  }
}
```
- **Response (201 Created):** Comment object with `$URN` in the `x-restli-id` header.

**Get Comments on a Post**
```
GET /rest/socialActions/{postUrn}/comments
```
- **Query Parameters:**
  - `start` -- Pagination start index (0-based)
  - `count` -- Number of results (max 100)
- **Response (200 OK):**
```json
{
  "elements": [
    {
      "actor": "urn:li:person:{personId}",
      "message": { "text": "Great post!" },
      "created": { "time": 1700000000000 },
      "$URN": "urn:li:comment:({postUrn},{commentId})"
    }
  ],
  "paging": { "start": 0, "count": 10, "total": 25 }
}
```

**Delete a Comment**
```
DELETE /rest/socialActions/{postUrn}/comments/{commentId}
```
- **Response:** `204 No Content`

#### Reactions

**React to a Post**
```
POST /rest/socialActions/{postUrn}/likes
```
- **Request Body:**
```json
{
  "actor": "urn:li:person:{personId}",
  "object": "{postUrn}"
}
```
- **Note:** LinkedIn now supports multiple reaction types (LIKE, CELEBRATE, SUPPORT, LOVE, INSIGHTFUL, FUNNY). Use the `reactionType` field when available.

**Get Reactions on a Post**
```
GET /rest/socialActions/{postUrn}/likes
```
- **Query Parameters:** `start`, `count`
- **Response:** Array of reaction objects with `actor`, `created.time`, and `reactionType`.

**Remove a Reaction**
```
DELETE /rest/socialActions/{postUrn}/likes/{actorUrn}
```
- **Response:** `204 No Content`

#### Organization Management

**Get Organization by ID**
```
GET /rest/organizations/{organizationId}
```
- **Response (200 OK):**
```json
{
  "id": 12345678,
  "localizedName": "Esmer Inc.",
  "vanityName": "esmer-inc",
  "logoV2": {
    "original": "urn:li:digitalmediaAsset:...",
    "cropped": "urn:li:digitalmediaAsset:..."
  },
  "localizedDescription": "AI delegation assistant",
  "localizedWebsite": "https://esmer.app",
  "staffCountRange": "SIZE_11_50",
  "industries": ["urn:li:industry:4"]
}
```

**Get Organization Followers Count**
```
GET /rest/networkSizes/{organizationUrn}?edgeType=CompanyFollowedByMember
```
- **Response (200 OK):**
```json
{
  "firstDegreeSize": 15000
}
```

**Get Organization Posts**
```
GET /rest/posts?author={organizationUrn}&q=author
```
- **Query Parameters:** `start`, `count`, `q=author`
- **Response:** Paginated list of post objects.

#### Analytics

**Get Share Statistics (Post Analytics)**
```
GET /rest/organizationalEntityShareStatistics?q=organizationalEntity&organizationalEntity={orgUrn}
```
- **Query Parameters:**
  - `q=organizationalEntity`
  - `organizationalEntity` -- `urn:li:organization:{id}`
  - `timeIntervals.timeGranularityType` -- `DAY` or `MONTH`
  - `timeIntervals.timeRange.start` -- Epoch milliseconds
  - `timeIntervals.timeRange.end` -- Epoch milliseconds
  - `shares[]` -- Specific share URNs (optional)
- **Response (200 OK):**
```json
{
  "elements": [
    {
      "totalShareStatistics": {
        "shareCount": 5,
        "clickCount": 120,
        "engagement": 0.045,
        "likeCount": 85,
        "impressionCount": 3500,
        "commentCount": 12,
        "shareCount": 8
      },
      "organizationalEntity": "urn:li:organization:12345678",
      "timeRange": {
        "start": 1700000000000,
        "end": 1700086400000
      }
    }
  ]
}
```

**Get Follower Statistics**
```
GET /rest/organizationalEntityFollowerStatistics?q=organizationalEntity&organizationalEntity={orgUrn}
```
- **Response:** Follower demographics including counts by seniority, industry, function, geo, and company size.

**Get Page Statistics**
```
GET /rest/organizationPageStatistics?q=organization&organization={orgUrn}
```
- **Response:** Page views by section (overview, jobs, people, etc.)

#### Profiles

**Get Authenticated User Profile**
```
GET /rest/me
```
- **Response (200 OK):**
```json
{
  "id": "{personId}",
  "localizedFirstName": "Jane",
  "localizedLastName": "Doe",
  "profilePicture": {
    "displayImage": "urn:li:digitalmediaAsset:..."
  },
  "vanityName": "janedoe"
}
```

**Get Profile Picture URLs**
```
GET /rest/me?projection=(id,profilePicture(displayImage~:playableStreams))
```
- **Response:** Contains `elements` array with `identifiers[].identifier` URLs at different resolutions.

### 2.6 Webhooks / Real-time

LinkedIn does not provide a general-purpose webhook API for content events. Real-time capabilities are limited:

- **No webhooks** for new comments, reactions, or followers.
- **Polling Required:** Esmer must poll the comments and analytics endpoints periodically (recommended: every 5-15 minutes for active posts, every hour for organization analytics).
- **LinkedIn Webhooks (limited):** Available only for specific enterprise integrations (e.g., Apply Connect for job applications). Not applicable to general social media management.

**Esmer Strategy:**
- Implement a polling service that checks for new comments and reactions on recent posts.
- Use the `createdAt` timestamp to detect new items since last poll.
- Cache organization analytics and refresh every 15-30 minutes.

### 2.7 Error Handling

| HTTP Code | Error Type | Description | Retry? |
|---|---|---|---|
| 400 | `Bad Request` | Malformed request, invalid URN format | No -- fix request |
| 401 | `Unauthorized` | Token expired or invalid | Yes -- refresh token |
| 403 | `Forbidden` | Insufficient permissions, app review not completed | No -- check scopes |
| 404 | `Not Found` | Resource does not exist | No |
| 409 | `Conflict` | Duplicate post within short window | No |
| 422 | `Unprocessable Entity` | Invalid field values, URN mismatch | No -- fix request |
| 429 | `Too Many Requests` | Rate limit exceeded | Yes -- wait per `Retry-After` |
| 500 | `Internal Server Error` | LinkedIn server error | Yes -- exponential backoff |
| 502 | `Bad Gateway` | LinkedIn infrastructure issue | Yes -- exponential backoff |
| 503 | `Service Unavailable` | LinkedIn temporarily unavailable | Yes -- exponential backoff |

**Error Response Format:**
```json
{
  "status": 403,
  "serviceErrorCode": 100,
  "code": "ACCESS_DENIED",
  "message": "Not enough permissions to access: POST /rest/posts"
}
```

### 2.8 Mobile-Specific Notes

- **Media Upload from Device:** LinkedIn requires a two-step upload process (initialize + PUT binary). Compress images to JPEG quality 85, max 10 MB. For video, compress to H.264 MP4 at 720p before uploading. Use the chunked upload for videos to handle interruptions.
- **Organization Selector:** If the user manages multiple LinkedIn pages, present an organization picker. Fetch the user's admin organizations via `GET /rest/organizationAcls?q=roleAssignee&role=ADMINISTRATOR&projection=(elements*(organization~(localizedName,logoV2)))`.
- **Post Preview:** Render a LinkedIn-style card preview showing the post text (3,000 char max indicator), attached link card with title/description/thumbnail, or image/video preview.
- **Content Formatting:** LinkedIn supports limited formatting in posts via the API: line breaks (`\n`) are preserved. Hashtags are automatically linked. @mentions require using the `{urn:li:person:ID}` format in the commentary text.
- **Character Limit Display:** Show a 3,000-character counter. LinkedIn does not support threads, so if content exceeds the limit, suggest splitting into a post + article.
- **Analytics Dashboard:** Cache analytics data locally and display charts for impressions, engagement rate, clicks, and follower growth over the selected time period.

---

## 3. Facebook Graph API (Meta Graph API)

### 3.1 Service Overview

The Facebook Graph API (Meta Graph API) provides programmatic access to Facebook Pages, user profiles, posts, photos, videos, comments, and insights. Esmer integrates with the Graph API to enable users to manage Facebook Pages (publish posts, respond to comments, upload media), retrieve page analytics and insights, monitor engagement, and manage ad campaigns. The API uses a node-and-edge model where every object (user, page, post, photo) is a node with properties and connections (edges) to other nodes. Esmer primarily targets Page management use cases since user-level posting is restricted by Meta's platform policies.

### 3.2 Authentication

**Method:** OAuth 2.0 (Authorization Code Flow) with App Access Tokens and Page Tokens

| Parameter | Value |
|---|---|
| Authorization URL | `https://www.facebook.com/v21.0/dialog/oauth` |
| Token URL | `https://graph.facebook.com/v21.0/oauth/access_token` |
| Grant Type | `authorization_code` |

**Token Types:**

| Token Type | Purpose | Expiry |
|---|---|---|
| App Access Token | Server-to-server calls, managing webhooks, reading public data | Never expires (tied to app) |
| User Access Token (short-lived) | User-specific data access | ~1-2 hours |
| User Access Token (long-lived) | Extended user access | ~60 days |
| Page Access Token (short-lived) | Manage a specific Page | ~1-2 hours |
| Page Access Token (long-lived) | Extended Page management | Never expires (if exchanged from long-lived user token) |

**Token Exchange Flow:**
1. User authorizes app, receive short-lived user token
2. Exchange for long-lived user token: `GET /oauth/access_token?grant_type=fb_exchange_token&client_id={app_id}&client_secret={app_secret}&fb_exchange_token={short_lived_token}`
3. Get Page token from long-lived user token: `GET /{user_id}/accounts?access_token={long_lived_user_token}`
4. The resulting Page token is long-lived (never expires) when derived from a long-lived user token.

**Required Permissions (Scopes):**

| Permission | Purpose | App Review Required? |
|---|---|---|
| `pages_show_list` | List pages the user manages | Yes |
| `pages_read_engagement` | Read page posts, comments, reactions | Yes |
| `pages_manage_posts` | Create, edit, delete page posts | Yes |
| `pages_manage_engagement` | Manage comments and reactions on page posts | Yes |
| `pages_manage_metadata` | Manage page settings and metadata | Yes |
| `pages_read_user_content` | Read user-generated content on page | Yes |
| `pages_manage_ads` | Manage page ads | Yes |
| `pages_manage_instant_articles` | Manage Instant Articles | Yes |
| `publish_video` | Upload videos to pages | Yes |
| `read_insights` | Read page and post insights/analytics | Yes |
| `business_management` | Manage business assets | Yes |
| `instagram_basic` | Read Instagram profile and media | Yes |
| `instagram_content_publish` | Publish to Instagram | Yes |
| `public_profile` | Read user's public profile | No |
| `email` | Read user's email | No |

**Recommendations for Esmer:**
- Use long-lived Page Tokens for persistent Page management. Store them securely server-side.
- All Pages-related permissions require App Review by Meta. Submit your app for review with detailed descriptions and screencasts.
- Generate an App Access Token for webhook verification: `{app_id}|{app_secret}` (concatenated).
- Use the `appsecret_proof` parameter (HMAC-SHA256 of access token using app secret) for all server-to-server calls to prevent token hijacking.

### 3.3 Base URL

```
https://graph.facebook.com/v21.0
```

Video upload endpoint:
```
https://graph-video.facebook.com/v21.0
```

### 3.4 Rate Limits

Meta uses a complex rate limiting system with different limits for different token types.

**Application-Level Rate Limiting:**

| Metric | Limit |
|---|---|
| Calls per hour | 200 calls/user/hour (aggregated across all users of the app) |
| Calculation | `200 * {number_of_users}` calls per hour for the app |
| Burst protection | Dynamic throttling based on error rates |

**Page-Level Rate Limiting:**

| Metric | Limit |
|---|---|
| Reads | 4,800 calls / 24-hour rolling window per Page |
| Publishes | 250 posts / 24-hour rolling window per Page |
| Videos | 1,000 uploads / 24-hour rolling window per Page |
| Comments | Not explicitly capped, but subject to spam detection |

**Business Use Case Rate Limiting (BUC):**
- Applies to apps that have completed App Review
- Higher limits based on the approved use case
- Specific limits communicated during App Review approval

**Rate Limit Headers:**
- `X-App-Usage` -- JSON: `{ "call_count": 50, "total_cputime": 25, "total_time": 35 }` (percentages of limit)
- `X-Page-Usage` -- JSON with the same structure for Page-level limits
- `x-business-use-case-usage` -- JSON with BUC-specific usage

**Throttle Response:** `429 Too Many Requests` or errors embedded in the `200 OK` response body with error code `4` (application-level) or `32` (Page-level).

### 3.5 API Endpoints

#### Pages

**Get Pages Managed by User**
```
GET /me/accounts
```
- **Description:** List all Facebook Pages the authenticated user manages.
- **Token:** User Access Token with `pages_show_list`
- **Query Parameters:**
  - `fields` -- `id,name,access_token,category,fan_count,picture,cover,link,about,description,website`
- **Response (200 OK):**
```json
{
  "data": [
    {
      "id": "123456789",
      "name": "Esmer Official",
      "access_token": "EAA...longpagetokenhere",
      "category": "Software",
      "fan_count": 15000,
      "picture": {
        "data": {
          "url": "https://scontent.xx.fbcdn.net/..."
        }
      }
    }
  ],
  "paging": {
    "cursors": { "before": "...", "after": "..." },
    "next": "https://graph.facebook.com/v21.0/me/accounts?after=..."
  }
}
```

**Get Page Details**
```
GET /{page_id}
```
- **Token:** Page Access Token
- **Query Parameters:**
  - `fields` -- `id,name,about,description,category,fan_count,followers_count,link,picture,cover,website,phone,emails,location,hours,single_line_address,rating_count,overall_star_rating,were_here_count`
- **Response (200 OK):**
```json
{
  "id": "123456789",
  "name": "Esmer Official",
  "about": "AI delegation assistant",
  "fan_count": 15000,
  "followers_count": 14800,
  "category": "Software",
  "link": "https://www.facebook.com/esmerofficial",
  "website": "https://esmer.app"
}
```

#### Posts

**Create a Page Post (Text)**
```
POST /{page_id}/feed
```
- **Token:** Page Access Token with `pages_manage_posts`
- **Request Body (form or JSON):**
```json
{
  "message": "Exciting news from Esmer! Our latest update includes AI-powered task delegation.",
  "link": "https://esmer.app/blog/update",
  "published": true,
  "scheduled_publish_time": 1700000000
}
```
- **Parameters:**
  - `message` -- Post text content (max 63,206 characters)
  - `link` -- URL to share (will generate a link preview card)
  - `published` -- `true` for immediate publish, `false` for draft/scheduled
  - `scheduled_publish_time` -- Unix timestamp for scheduled post (10 min to 6 months in the future)
  - `targeting` -- JSON object for audience targeting (country, city, age, etc.)
  - `place` -- Place ID for location tagging
- **Response (200 OK):**
```json
{
  "id": "123456789_987654321"
}
```

**Create a Page Post with Photo**
```
POST /{page_id}/photos
```
- **Token:** Page Access Token with `pages_manage_posts`
- **Request Body (multipart/form-data):**
  - `source` -- Binary image file
  - `message` -- Caption text
  - `published` -- `true` or `false`
- **Alternatively (URL-based):**
```json
{
  "url": "https://example.com/image.jpg",
  "message": "Check out this image!",
  "published": true
}
```
- **Response (200 OK):**
```json
{
  "id": "photo_id",
  "post_id": "page_id_post_id"
}
```
- **Constraints:** Max image size 10 MB. Formats: JPEG, PNG, GIF, BMP, TIFF. Max 1,000 photos per post.

**Create a Multi-Photo Post**

Step 1 -- Upload each photo as unpublished:
```
POST /{page_id}/photos
```
```json
{ "url": "https://example.com/photo1.jpg", "published": false }
```
- Repeat for each photo. Collect the returned `id` values.

Step 2 -- Create the post referencing all photos:
```
POST /{page_id}/feed
```
```json
{
  "message": "Multiple photos from our event!",
  "attached_media[0]": { "media_fbid": "{photo1_id}" },
  "attached_media[1]": { "media_fbid": "{photo2_id}" },
  "attached_media[2]": { "media_fbid": "{photo3_id}" }
}
```

**Upload a Video**
```
POST https://graph-video.facebook.com/v21.0/{page_id}/videos
```
- **Token:** Page Access Token with `publish_video`
- **Simple Upload (< 1 GB):**
```
POST /{page_id}/videos
Content-Type: multipart/form-data

source={binary_video}&title=My Video&description=Video description
```
- **Resumable Upload (> 1 GB, up to 10 GB):**

Step 1 -- Start:
```
POST /{page_id}/videos
```
```json
{
  "upload_phase": "start",
  "file_size": 1073741824
}
```
- Response: `{ "upload_session_id": "...", "video_id": "...", "start_offset": "0", "end_offset": "52428800" }`

Step 2 -- Transfer (repeat for each chunk):
```
POST /{page_id}/videos
```
- Form data: `upload_phase=transfer`, `upload_session_id`, `start_offset`, `video_file_chunk` (binary)
- Response: next `start_offset` and `end_offset`

Step 3 -- Finish:
```
POST /{page_id}/videos
```
```json
{
  "upload_phase": "finish",
  "upload_session_id": "...",
  "title": "My Video",
  "description": "Video description"
}
```
- Response: `{ "success": true }`

- **Constraints:** Max 10 GB file size, max 4 hours duration. Formats: MP4 (recommended), MOV, AVI, WMV, FLV. Recommended: H.264, AAC audio, 720p+.

**Get Posts on a Page**
```
GET /{page_id}/feed
```
- **Token:** Page Access Token with `pages_read_engagement`
- **Query Parameters:**
  - `fields` -- `id,message,created_time,full_picture,permalink_url,shares,attachments{media,media_type,title,description,url},from,type,status_type,is_published`
  - `limit` -- Number of posts (max 100)
  - `since`, `until` -- Unix timestamps for date filtering
- **Response (200 OK):**
```json
{
  "data": [
    {
      "id": "123456789_987654321",
      "message": "Post content here",
      "created_time": "2026-02-08T12:00:00+0000",
      "full_picture": "https://scontent.xx.fbcdn.net/...",
      "permalink_url": "https://www.facebook.com/...",
      "shares": { "count": 15 },
      "from": { "name": "Esmer Official", "id": "123456789" }
    }
  ],
  "paging": {
    "cursors": { "before": "...", "after": "..." },
    "next": "..."
  }
}
```

**Get a Specific Post**
```
GET /{post_id}
```
- **Query Parameters:**
  - `fields` -- `id,message,created_time,full_picture,permalink_url,shares,likes.summary(true),comments.summary(true),reactions.summary(true),attachments,from,type`
- **Response:** Single post object with requested fields.

**Update a Post**
```
POST /{post_id}
```
- **Request Body:**
```json
{ "message": "Updated post content" }
```
- **Response:** `{ "success": true }`

**Delete a Post**
```
DELETE /{post_id}
```
- **Token:** Page Access Token with `pages_manage_posts`
- **Response:** `{ "success": true }`

#### Comments

**Get Comments on a Post**
```
GET /{post_id}/comments
```
- **Token:** Page Access Token with `pages_read_engagement`
- **Query Parameters:**
  - `fields` -- `id,message,created_time,from,like_count,comment_count,attachment,parent`
  - `filter` -- `toplevel` (only top-level) or `stream` (all including replies)
  - `order` -- `ranked` or `chronological`
  - `limit` -- Max 100
  - `summary` -- `true` to include total count
- **Response (200 OK):**
```json
{
  "data": [
    {
      "id": "987654321_111222333",
      "message": "Love this update!",
      "created_time": "2026-02-08T13:00:00+0000",
      "from": { "name": "John Doe", "id": "555666777" },
      "like_count": 5,
      "comment_count": 2
    }
  ],
  "paging": { "cursors": { "before": "...", "after": "..." } },
  "summary": { "total_count": 42, "can_comment": true }
}
```

**Reply to a Comment**
```
POST /{comment_id}/comments
```
- **Token:** Page Access Token with `pages_manage_engagement`
- **Request Body:**
```json
{ "message": "Thank you for your feedback! We appreciate it." }
```
- **Response:** `{ "id": "{new_comment_id}" }`

**Delete a Comment**
```
DELETE /{comment_id}
```
- **Response:** `{ "success": true }`

**Hide a Comment**
```
POST /{comment_id}
```
```json
{ "is_hidden": true }
```
- **Response:** `{ "success": true }`

#### Reactions

**Get Reactions on a Post**
```
GET /{post_id}/reactions
```
- **Query Parameters:**
  - `type` -- Filter by reaction type: `LIKE`, `LOVE`, `WOW`, `HAHA`, `SAD`, `ANGRY`, `CARE`
  - `summary` -- `total_count`
  - `limit` -- Max 100
- **Response (200 OK):**
```json
{
  "data": [
    { "id": "555666777", "name": "John Doe", "type": "LIKE" }
  ],
  "paging": { "cursors": { "before": "...", "after": "..." } },
  "summary": { "total_count": 250 }
}
```

#### Insights (Analytics)

**Get Page Insights**
```
GET /{page_id}/insights
```
- **Token:** Page Access Token with `read_insights`
- **Query Parameters:**
  - `metric` -- Comma-separated: `page_impressions,page_impressions_unique,page_engaged_users,page_post_engagements,page_fan_adds,page_fan_removes,page_fans,page_views_total,page_actions_post_reactions_total`
  - `period` -- `day`, `week`, `days_28`, `month`, `lifetime`
  - `since`, `until` -- Unix timestamps
  - `date_preset` -- `today`, `yesterday`, `this_week`, `last_week`, `this_month`
- **Response (200 OK):**
```json
{
  "data": [
    {
      "name": "page_impressions",
      "period": "day",
      "title": "Daily Total Impressions",
      "values": [
        { "value": 3500, "end_time": "2026-02-08T08:00:00+0000" },
        { "value": 4200, "end_time": "2026-02-09T08:00:00+0000" }
      ]
    }
  ]
}
```

**Get Post Insights**
```
GET /{post_id}/insights
```
- **Query Parameters:**
  - `metric` -- `post_impressions,post_impressions_unique,post_engaged_users,post_clicks,post_reactions_by_type_total,post_activity_by_action_type`
- **Response:** Same structure as page insights, scoped to a single post.

**Key Page Metrics:**

| Metric | Description |
|---|---|
| `page_impressions` | Total impressions for content associated with the Page |
| `page_impressions_unique` | Unique users who saw any Page content |
| `page_engaged_users` | Unique users who engaged with the Page |
| `page_post_engagements` | Total engagements with Page posts |
| `page_fan_adds` | New Page likes/follows |
| `page_fan_removes` | Page unlikes/unfollows |
| `page_fans` | Total Page likes (lifetime) |
| `page_views_total` | Total Page views |
| `page_actions_post_reactions_total` | Total reactions on Page posts |

**Key Post Metrics:**

| Metric | Description |
|---|---|
| `post_impressions` | Total impressions for the post |
| `post_impressions_unique` | Unique users who saw the post |
| `post_engaged_users` | Unique users who clicked the post |
| `post_clicks` | Total clicks on the post |
| `post_reactions_by_type_total` | Reactions broken down by type |

### 3.6 Webhooks / Real-time

Meta provides a robust Webhooks system for real-time notifications.

**Setup Webhooks:**

1. Configure in the Meta App Dashboard under the Webhooks product.
2. Provide a callback URL and a verify token.
3. Meta sends a `GET` challenge to verify the endpoint:
   - Query params: `hub.mode=subscribe`, `hub.challenge={challenge_string}`, `hub.verify_token={your_token}`
   - Respond with the `hub.challenge` value and `200 OK`.

**Subscribe to Page Events:**
```
POST /{page_id}/subscribed_apps
```
- **Token:** Page Access Token
- **Request Body:**
```json
{
  "subscribed_fields": "feed,messages,message_deliveries,message_reads,messaging_postbacks,messaging_optins,messaging_optouts,comments,mention"
}
```
- **Response:** `{ "success": true }`

**Webhook Event Payload (example -- new comment):**
```json
{
  "entry": [
    {
      "id": "123456789",
      "time": 1700000000,
      "changes": [
        {
          "field": "feed",
          "value": {
            "item": "comment",
            "comment_id": "987654321_111222333",
            "post_id": "123456789_987654321",
            "verb": "add",
            "message": "Great post!",
            "from": { "id": "555666777", "name": "John Doe" },
            "created_time": 1700000000
          }
        }
      ]
    }
  ],
  "object": "page"
}
```

**Subscribable Fields:**

| Field | Events |
|---|---|
| `feed` | New posts, comments, reactions, shares on the Page |
| `messages` | New messages to the Page (Messenger) |
| `message_deliveries` | Message delivery confirmations |
| `message_reads` | Message read receipts |
| `messaging_postbacks` | Postback button clicks |
| `mention` | Page mentioned in a post or comment |
| `ratings` | New Page reviews/ratings |
| `leadgen` | New lead from a Lead Ad |

**Security:** Validate webhook payloads by computing HMAC-SHA256 of the raw request body using the App Secret, and comparing with the `X-Hub-Signature-256` header.

### 3.7 Error Handling

| HTTP Code | Error Code | Description | Retry? |
|---|---|---|---|
| 400 | 100 | Invalid parameter | No -- fix request |
| 400 | 190 | Access token expired or invalid | Yes -- refresh token |
| 400 | 200-299 | Permissions errors (various) | No -- check permissions |
| 400 | 368 | Content blocked by policy | No -- modify content |
| 403 | 4 | Application-level rate limit | Yes -- reduce call frequency |
| 403 | 17 | User-level rate limit | Yes -- wait and retry |
| 403 | 32 | Page-level rate limit | Yes -- wait and retry |
| 404 | 803 | Object does not exist | No |
| 500 | 1 | Unknown error | Yes -- exponential backoff |
| 500 | 2 | Temporary service issue | Yes -- exponential backoff |

**Error Response Format:**
```json
{
  "error": {
    "message": "Unsupported post request. Object with ID '123' does not exist...",
    "type": "GraphMethodException",
    "code": 100,
    "error_subcode": 33,
    "fbtrace_id": "ABC123DEF456"
  }
}
```

**Page Token vs User Token Differences:**

| Aspect | User Token | Page Token |
|---|---|---|
| Scope | User's own profile and authorized data | Specific Page's data and management |
| Expiry (short-lived) | ~1-2 hours | ~1-2 hours |
| Expiry (long-lived) | ~60 days | Never expires (from long-lived user token) |
| Can post to Page | No (deprecated) | Yes |
| Can read Page insights | No | Yes (with `read_insights`) |
| Can manage comments | No | Yes (with `pages_manage_engagement`) |
| Rate limits | Per-user | Per-Page (4,800 reads/day) |

### 3.8 Mobile-Specific Notes

- **Media Upload from Device:** Use multipart/form-data for images directly from the device camera roll. Compress to JPEG quality 85 and resize to max 2048px on the longest side. For videos, use the resumable upload protocol to handle mobile network interruptions.
- **Link Preview:** When composing a post with a link, use the Graph API's URL scraping endpoint (`POST /?id={url}&scrape=true`) to force Meta to fetch or refresh the Open Graph metadata, then display a preview card.
- **Scheduled Posts:** Allow users to schedule posts for future times (10 minutes to 6 months ahead). Show the scheduled post queue by fetching `/{page_id}/scheduled_posts`.
- **Multi-Page Management:** Many users manage multiple Pages. Present a Page switcher in the UI, caching each Page's access token securely.
- **Comment Moderation:** Provide a unified comment feed across all managed Pages with options to reply, hide, or delete. Use webhooks for real-time comment notifications.
- **Image Optimization:** Facebook re-compresses all uploaded images. For best quality, upload at least 1200x630px for link shares and 1080x1080px for photo posts. PNG format preserves quality better than JPEG for graphics with text.
- **Video Thumbnails:** Use `/{video_id}/thumbnails` to fetch available thumbnails after upload, or upload a custom thumbnail with `/{video_id}/thumbnails` POST.

---

## 4. YouTube (YouTube Data API v3)

### 4.1 Service Overview

YouTube is the world's largest video platform. Esmer integrates with the YouTube Data API v3 to enable users to manage their YouTube channels, upload and manage videos, create and manage playlists, moderate comments, and retrieve channel and video analytics. YouTube uses a quota-based system where each API operation consumes a specific number of quota units from a daily allocation. This makes careful quota management critical for Esmer's implementation.

### 4.2 Authentication

**Method:** OAuth 2.0 (Authorization Code Flow via Google)

| Parameter | Value |
|---|---|
| Authorization URL | `https://accounts.google.com/o/oauth2/v2/auth` |
| Token URL | `https://oauth2.googleapis.com/token` |
| Revoke URL | `https://oauth2.googleapis.com/revoke` |
| Grant Type | `authorization_code` |

**Required Scopes:**

| Scope | Purpose |
|---|---|
| `https://www.googleapis.com/auth/youtube` | Full access to manage YouTube account |
| `https://www.googleapis.com/auth/youtube.readonly` | Read-only access to YouTube data |
| `https://www.googleapis.com/auth/youtube.upload` | Upload videos |
| `https://www.googleapis.com/auth/youtube.force-ssl` | Manage YouTube data (required for comments, captions) |
| `https://www.googleapis.com/auth/youtubepartner` | YouTube Partner features (Content ID, etc.) |
| `https://www.googleapis.com/auth/yt-analytics.readonly` | Read YouTube Analytics data |
| `https://www.googleapis.com/auth/yt-analytics-monetary.readonly` | Read monetary YouTube Analytics data |

**Recommendations for Esmer:**
- Use `https://www.googleapis.com/auth/youtube` for full channel management.
- Use `https://www.googleapis.com/auth/youtube.readonly` for users who only want to view analytics.
- Combine with `https://www.googleapis.com/auth/yt-analytics.readonly` for analytics dashboards.
- Google OAuth tokens expire after 1 hour. Always use refresh tokens (`access_type=offline`).
- Request `prompt=consent` on first authorization to ensure a refresh token is returned.

### 4.3 Base URL

```
https://www.googleapis.com/youtube/v3
```

Upload endpoint:
```
https://www.googleapis.com/upload/youtube/v3/videos
```

Analytics endpoint:
```
https://youtubeanalytics.googleapis.com/v2
```

### 4.4 Rate Limits

YouTube uses a quota system rather than per-second rate limits. Each project gets 10,000 quota units per day by default.

**Quota Costs by Operation:**

| Operation | Quota Cost |
|---|---|
| `videos.list` | 1 unit |
| `channels.list` | 1 unit |
| `playlists.list` | 1 unit |
| `playlistItems.list` | 1 unit |
| `search.list` | 100 units |
| `comments.list` | 1 unit |
| `commentThreads.list` | 1 unit |
| `subscriptions.list` | 1 unit |
| `videoCategories.list` | 1 unit |
| `videos.insert` (upload) | 1,600 units |
| `videos.update` | 50 units |
| `videos.delete` | 50 units |
| `videos.rate` | 50 units |
| `playlists.insert` | 50 units |
| `playlists.update` | 50 units |
| `playlists.delete` | 50 units |
| `playlistItems.insert` | 50 units |
| `playlistItems.update` | 50 units |
| `playlistItems.delete` | 50 units |
| `comments.insert` | 50 units |
| `comments.update` | 50 units |
| `comments.delete` | 50 units |
| `comments.markAsSpam` | 50 units |
| `comments.setModerationStatus` | 50 units |
| `channels.update` | 50 units |
| `thumbnails.set` | 50 units |
| `channelBanners.insert` | 50 units |
| `watermarks.set` | 50 units |
| `watermarks.unset` | 50 units |

**Daily Quota:** 10,000 units by default. Can request increases via Google Cloud Console (requires justification and may take weeks for approval). Typical approved quotas range from 50,000 to 1,000,000 units.

**Practical Implications for Esmer:**
- A single video upload (1,600 units) uses 16% of the default daily quota.
- A search operation (100 units) uses 1% of the daily quota.
- Listing operations are cheap (1 unit each) -- batch these freely.
- Cache search results aggressively to minimize quota consumption.

**Error on Quota Exceeded:** `403 Forbidden` with `reason: quotaExceeded` or `dailyLimitExceeded`.

### 4.5 API Endpoints

#### Channels

**Get Channel Details**
```
GET /youtube/v3/channels
```
- **Quota Cost:** 1 unit
- **Query Parameters:**
  - `part` (required) -- `snippet,contentDetails,statistics,brandingSettings,status,topicDetails` (comma-separated)
  - `mine=true` -- Authenticated user's channel
  - `id` -- Channel ID (for looking up other channels)
  - `forUsername` -- Legacy username lookup
- **Response (200 OK):**
```json
{
  "kind": "youtube#channelListResponse",
  "items": [
    {
      "kind": "youtube#channel",
      "id": "UC1234567890abcdef",
      "snippet": {
        "title": "Esmer Channel",
        "description": "Official Esmer YouTube channel",
        "customUrl": "@esmerchannel",
        "publishedAt": "2020-01-01T00:00:00Z",
        "thumbnails": {
          "default": { "url": "https://yt3.ggpht.com/...", "width": 88, "height": 88 },
          "medium": { "url": "https://yt3.ggpht.com/...", "width": 240, "height": 240 },
          "high": { "url": "https://yt3.ggpht.com/...", "width": 800, "height": 800 }
        },
        "country": "US"
      },
      "statistics": {
        "viewCount": "1500000",
        "subscriberCount": "25000",
        "hiddenSubscriberCount": false,
        "videoCount": "150"
      },
      "contentDetails": {
        "relatedPlaylists": {
          "likes": "LL",
          "uploads": "UU1234567890abcdef"
        }
      },
      "brandingSettings": {
        "channel": {
          "title": "Esmer Channel",
          "description": "Official Esmer YouTube channel",
          "keywords": "esmer ai delegation assistant",
          "unsubscribedTrailer": "video_id_here"
        }
      }
    }
  ]
}
```

**Update Channel Details**
```
PUT /youtube/v3/channels
```
- **Quota Cost:** 50 units
- **Query Parameters:** `part=brandingSettings` (or other parts to update)
- **Request Body:**
```json
{
  "id": "UC1234567890abcdef",
  "brandingSettings": {
    "channel": {
      "description": "Updated channel description",
      "keywords": "esmer ai assistant mobile"
    }
  }
}
```
- **Response:** Updated channel resource.

**Upload Channel Banner**
```
POST /upload/youtube/v3/channelBanners/insert
```
- **Quota Cost:** 50 units
- **Content-Type:** `application/octet-stream` or `image/jpeg` or `image/png`
- **Body:** Raw binary image data
- **Constraints:** Recommended 2560x1440px, max 6 MB
- **Response:**
```json
{
  "kind": "youtube#channelBannerResource",
  "url": "https://www.googleapis.com/youtube/v3/channelBanners/..."
}
```
- After upload, update the channel's `brandingSettings.image.bannerExternalUrl` with the returned URL.

#### Videos

**Upload a Video**
```
POST /upload/youtube/v3/videos?uploadType=resumable&part=snippet,status
```
- **Quota Cost:** 1,600 units
- **Step 1 -- Initiate resumable upload:**
  - **Headers:**
    - `Authorization: Bearer {token}`
    - `Content-Type: application/json; charset=UTF-8`
    - `X-Upload-Content-Type: video/mp4`
    - `X-Upload-Content-Length: {total_bytes}`
  - **Request Body:**
```json
{
  "snippet": {
    "title": "My Video Title",
    "description": "Video description with #hashtags",
    "tags": ["esmer", "ai", "productivity"],
    "categoryId": "28",
    "defaultLanguage": "en"
  },
  "status": {
    "privacyStatus": "public",
    "selfDeclaredMadeForKids": false,
    "publishAt": "2026-02-15T12:00:00Z",
    "license": "youtube",
    "embeddable": true
  }
}
```
  - **Response:** `200 OK` with `Location` header containing the resumable upload URI.

- **Step 2 -- Upload video data:**
  - `PUT {upload_uri}`
  - **Headers:** `Content-Type: video/mp4`, `Content-Length: {chunk_size}`
  - **Body:** Binary video chunk
  - For resumable: upload in chunks (recommended 5-10 MB per chunk)
  - **Response (200 OK on completion):**
```json
{
  "kind": "youtube#video",
  "id": "dQw4w9WgXcQ",
  "snippet": {
    "title": "My Video Title",
    "channelId": "UC1234567890abcdef",
    "publishedAt": "2026-02-09T12:00:00Z",
    "thumbnails": {
      "default": { "url": "https://i.ytimg.com/vi/dQw4w9WgXcQ/default.jpg" },
      "medium": { "url": "https://i.ytimg.com/vi/dQw4w9WgXcQ/mqdefault.jpg" },
      "high": { "url": "https://i.ytimg.com/vi/dQw4w9WgXcQ/hqdefault.jpg" }
    }
  },
  "status": {
    "privacyStatus": "public",
    "uploadStatus": "uploaded"
  }
}
```
- **Constraints:** Max 256 GB or 12 hours (whichever is less). Default max without verification: 15 minutes. Formats: MP4, MOV, AVI, WMV, FLV, 3GP, MPEG4, WebM.

**Get Video Details**
```
GET /youtube/v3/videos
```
- **Quota Cost:** 1 unit
- **Query Parameters:**
  - `part` (required) -- `snippet,contentDetails,statistics,status,player,topicDetails,recordingDetails,liveStreamingDetails`
  - `id` -- Comma-separated video IDs (max 50)
  - `chart` -- `mostPopular` (for trending videos)
  - `myRating` -- `like` or `dislike` (authenticated user's rated videos)
  - `regionCode` -- ISO 3166-1 alpha-2 country code
- **Response (200 OK):**
```json
{
  "items": [
    {
      "id": "dQw4w9WgXcQ",
      "snippet": {
        "title": "Video Title",
        "description": "Full video description",
        "publishedAt": "2026-02-09T12:00:00Z",
        "channelId": "UC1234567890abcdef",
        "channelTitle": "Esmer Channel",
        "tags": ["esmer", "ai"],
        "categoryId": "28",
        "thumbnails": {
          "default": { "url": "...", "width": 120, "height": 90 },
          "medium": { "url": "...", "width": 320, "height": 180 },
          "high": { "url": "...", "width": 480, "height": 360 },
          "standard": { "url": "...", "width": 640, "height": 480 },
          "maxres": { "url": "...", "width": 1280, "height": 720 }
        }
      },
      "contentDetails": {
        "duration": "PT15M30S",
        "dimension": "2d",
        "definition": "hd",
        "caption": "true",
        "licensedContent": false
      },
      "statistics": {
        "viewCount": "150000",
        "likeCount": "5000",
        "commentCount": "320"
      },
      "status": {
        "uploadStatus": "processed",
        "privacyStatus": "public",
        "embeddable": true,
        "madeForKids": false
      }
    }
  ]
}
```

**Update Video Metadata**
```
PUT /youtube/v3/videos?part=snippet,status
```
- **Quota Cost:** 50 units
- **Request Body:**
```json
{
  "id": "dQw4w9WgXcQ",
  "snippet": {
    "title": "Updated Title",
    "description": "Updated description",
    "tags": ["updated", "tags"],
    "categoryId": "28"
  },
  "status": {
    "privacyStatus": "unlisted"
  }
}
```
- **Response:** Updated video resource.

**Delete a Video**
```
DELETE /youtube/v3/videos?id={videoId}
```
- **Quota Cost:** 50 units
- **Response:** `204 No Content`

**Rate a Video**
```
POST /youtube/v3/videos/rate?id={videoId}&rating={rating}
```
- **Quota Cost:** 50 units
- **Parameters:**
  - `id` -- Video ID
  - `rating` -- `like`, `dislike`, or `none`
- **Response:** `204 No Content`

**Search Videos**
```
GET /youtube/v3/search
```
- **Quota Cost:** 100 units (expensive -- cache results)
- **Query Parameters:**
  - `part=snippet` (required)
  - `q` -- Search query
  - `type` -- `video`, `channel`, `playlist`
  - `maxResults` -- 0-50 (default 5)
  - `pageToken` -- Pagination token
  - `order` -- `relevance`, `date`, `rating`, `viewCount`, `title`
  - `publishedAfter`, `publishedBefore` -- ISO 8601
  - `channelId` -- Filter by channel
  - `regionCode` -- ISO 3166-1 alpha-2
  - `videoDuration` -- `any`, `short` (<4 min), `medium` (4-20 min), `long` (>20 min)
  - `videoDefinition` -- `any`, `high`, `standard`
  - `videoType` -- `any`, `episode`, `movie`
  - `safeSearch` -- `none`, `moderate`, `strict`
- **Response (200 OK):**
```json
{
  "items": [
    {
      "kind": "youtube#searchResult",
      "id": { "kind": "youtube#video", "videoId": "dQw4w9WgXcQ" },
      "snippet": {
        "title": "Video Title",
        "description": "Truncated description...",
        "publishedAt": "2026-02-09T12:00:00Z",
        "channelId": "UC1234567890abcdef",
        "channelTitle": "Esmer Channel",
        "thumbnails": { "default": {}, "medium": {}, "high": {} }
      }
    }
  ],
  "pageInfo": { "totalResults": 1000000, "resultsPerPage": 5 },
  "nextPageToken": "CAUQAA"
}
```

#### Playlists

**Create a Playlist**
```
POST /youtube/v3/playlists?part=snippet,status
```
- **Quota Cost:** 50 units
- **Request Body:**
```json
{
  "snippet": {
    "title": "My Esmer Playlist",
    "description": "Curated videos about AI delegation",
    "tags": ["esmer", "playlist"],
    "defaultLanguage": "en"
  },
  "status": {
    "privacyStatus": "public"
  }
}
```
- **Response:** Playlist resource with `id`.

**Get Playlists**
```
GET /youtube/v3/playlists
```
- **Quota Cost:** 1 unit
- **Query Parameters:**
  - `part=snippet,contentDetails,status`
  - `mine=true` or `channelId={id}` or `id={playlist_id}`
  - `maxResults` -- 0-50
  - `pageToken`

**Update a Playlist**
```
PUT /youtube/v3/playlists?part=snippet,status
```
- **Quota Cost:** 50 units
- **Request Body:** Full playlist resource with `id` and updated fields.

**Delete a Playlist**
```
DELETE /youtube/v3/playlists?id={playlistId}
```
- **Quota Cost:** 50 units
- **Response:** `204 No Content`

#### Playlist Items

**Add Video to Playlist**
```
POST /youtube/v3/playlistItems?part=snippet
```
- **Quota Cost:** 50 units
- **Request Body:**
```json
{
  "snippet": {
    "playlistId": "PLxxxxxxxxxxxxxxxxx",
    "resourceId": {
      "kind": "youtube#video",
      "videoId": "dQw4w9WgXcQ"
    },
    "position": 0
  }
}
```
- **Response:** PlaylistItem resource.

**Get Playlist Items**
```
GET /youtube/v3/playlistItems
```
- **Quota Cost:** 1 unit
- **Query Parameters:**
  - `part=snippet,contentDetails,status`
  - `playlistId` (required)
  - `maxResults` -- 0-50
  - `pageToken`
  - `videoId` -- Filter for a specific video

**Remove Video from Playlist**
```
DELETE /youtube/v3/playlistItems?id={playlistItemId}
```
- **Quota Cost:** 50 units
- **Note:** Use the `playlistItemId` (not the `videoId`).

#### Comments

**Get Comment Threads for a Video**
```
GET /youtube/v3/commentThreads
```
- **Quota Cost:** 1 unit
- **Query Parameters:**
  - `part=snippet,replies`
  - `videoId` -- Video ID
  - `allThreadsRelatedToChannelId` -- All comments on any video in the channel
  - `maxResults` -- 1-100 (default 20)
  - `pageToken`
  - `order` -- `time` or `relevance`
  - `moderationStatus` -- `published`, `heldForReview`, `likelySpam`, `rejected`
  - `searchTerms` -- Filter by keyword
- **Response (200 OK):**
```json
{
  "items": [
    {
      "kind": "youtube#commentThread",
      "id": "comment_thread_id",
      "snippet": {
        "videoId": "dQw4w9WgXcQ",
        "topLevelComment": {
          "kind": "youtube#comment",
          "id": "comment_id",
          "snippet": {
            "videoId": "dQw4w9WgXcQ",
            "textDisplay": "Great video!",
            "textOriginal": "Great video!",
            "authorDisplayName": "John Doe",
            "authorProfileImageUrl": "https://...",
            "authorChannelUrl": "https://...",
            "likeCount": 5,
            "publishedAt": "2026-02-08T12:00:00Z",
            "updatedAt": "2026-02-08T12:00:00Z"
          }
        },
        "canReply": true,
        "totalReplyCount": 2,
        "isPublic": true
      },
      "replies": {
        "comments": [
          {
            "snippet": {
              "textDisplay": "Thanks!",
              "authorDisplayName": "Esmer Channel",
              "parentId": "comment_id"
            }
          }
        ]
      }
    }
  ]
}
```

**Post a Comment**
```
POST /youtube/v3/commentThreads?part=snippet
```
- **Quota Cost:** 50 units
- **Request Body:**
```json
{
  "snippet": {
    "videoId": "dQw4w9WgXcQ",
    "topLevelComment": {
      "snippet": {
        "textOriginal": "Awesome content! Keep it up."
      }
    }
  }
}
```

**Reply to a Comment**
```
POST /youtube/v3/comments?part=snippet
```
- **Quota Cost:** 50 units
- **Request Body:**
```json
{
  "snippet": {
    "parentId": "parent_comment_id",
    "textOriginal": "Thanks for watching!"
  }
}
```

**Update a Comment**
```
PUT /youtube/v3/comments?part=snippet
```
- **Quota Cost:** 50 units
- **Request Body:**
```json
{
  "id": "comment_id",
  "snippet": {
    "textOriginal": "Updated comment text"
  }
}
```

**Delete a Comment**
```
DELETE /youtube/v3/comments?id={commentId}
```
- **Quota Cost:** 50 units

**Moderate a Comment**
```
POST /youtube/v3/comments/setModerationStatus?id={commentId}&moderationStatus={status}
```
- **Quota Cost:** 50 units
- **Parameters:**
  - `moderationStatus` -- `published`, `heldForReview`, `rejected`
  - `banAuthor` -- `true` to ban the commenter from the channel (only with `rejected`)

#### Video Categories

**List Video Categories**
```
GET /youtube/v3/videoCategories
```
- **Quota Cost:** 1 unit
- **Query Parameters:**
  - `part=snippet`
  - `regionCode` -- ISO 3166-1 alpha-2 (e.g., `US`)
  - `id` -- Specific category IDs
- **Common Categories:**

| ID | Name |
|---|---|
| 1 | Film & Animation |
| 2 | Autos & Vehicles |
| 10 | Music |
| 15 | Pets & Animals |
| 17 | Sports |
| 20 | Gaming |
| 22 | People & Blogs |
| 23 | Comedy |
| 24 | Entertainment |
| 25 | News & Politics |
| 26 | Howto & Style |
| 27 | Education |
| 28 | Science & Technology |

#### Thumbnails

**Set Custom Thumbnail**
```
POST /upload/youtube/v3/thumbnails/set?videoId={videoId}
```
- **Quota Cost:** 50 units
- **Content-Type:** `image/jpeg`, `image/png`, or `image/gif`
- **Body:** Raw binary image data
- **Constraints:** Recommended 1280x720px, max 2 MB. Channel must be verified to upload custom thumbnails.
- **Response:**
```json
{
  "items": [
    {
      "default": { "url": "...", "width": 120, "height": 90 },
      "medium": { "url": "...", "width": 320, "height": 180 },
      "high": { "url": "...", "width": 480, "height": 360 }
    }
  ]
}
```

#### Analytics (YouTube Analytics API)

**Query Analytics**
```
GET https://youtubeanalytics.googleapis.com/v2/reports
```
- **Query Parameters:**
  - `ids=channel==MINE` or `ids=channel=={channelId}`
  - `startDate` -- `YYYY-MM-DD`
  - `endDate` -- `YYYY-MM-DD`
  - `metrics` -- `views,estimatedMinutesWatched,averageViewDuration,averageViewPercentage,likes,dislikes,comments,shares,subscribersGained,subscribersLost`
  - `dimensions` -- `day`, `video`, `country`, `ageGroup`, `gender`, `deviceType`, `operatingSystem`, `liveOrOnDemand`, `trafficSource`
  - `filters` -- `video=={videoId}`, `country=={countryCode}`
  - `sort` -- e.g., `-views` (descending)
  - `maxResults` -- Max 200
- **Response (200 OK):**
```json
{
  "kind": "youtubeAnalytics#resultTable",
  "columnHeaders": [
    { "name": "day", "columnType": "DIMENSION", "dataType": "STRING" },
    { "name": "views", "columnType": "METRIC", "dataType": "INTEGER" },
    { "name": "estimatedMinutesWatched", "columnType": "METRIC", "dataType": "FLOAT" },
    { "name": "likes", "columnType": "METRIC", "dataType": "INTEGER" }
  ],
  "rows": [
    ["2026-02-08", 1500, 4500.0, 85],
    ["2026-02-09", 2200, 6600.0, 120]
  ]
}
```

### 4.6 Webhooks / Real-time

YouTube does not offer traditional webhooks. Real-time update options include:

**PubSubHubbub (WebSub) for New Video Notifications:**
```
POST https://pubsubhubbub.appspot.com/subscribe
```
- **Form Parameters:**
  - `hub.callback` -- Your webhook URL
  - `hub.topic` -- `https://www.youtube.com/xml/feeds/videos.xml?channel_id={channelId}`
  - `hub.verify` -- `sync` or `async`
  - `hub.mode` -- `subscribe` or `unsubscribe`
  - `hub.lease_seconds` -- Subscription duration (max ~10 days, must renew)
- **Verification:** YouTube sends a `GET` to `hub.callback` with `hub.challenge`. Respond with the challenge value.
- **Notification Payload:** Atom XML feed with the new video entry.
- **Limitations:** Only notifies about new public video uploads, not edits, deletions, or comments.

**Polling Strategy for Esmer:**
- For new videos: Use PubSubHubbub subscriptions with auto-renewal.
- For comments: Poll `commentThreads.list` every 5-10 minutes for active videos (1 quota unit per call).
- For analytics: Poll the Analytics API once per hour (analytics data has a 24-48 hour delay).
- For video status: After upload, poll `videos.list` with the video ID every 30 seconds until `status.uploadStatus` is `processed`.

### 4.7 Error Handling

| HTTP Code | Error Reason | Description | Retry? |
|---|---|---|---|
| 400 | `badRequest` | Invalid request parameters | No -- fix request |
| 400 | `mediaBodyRequired` | Missing video file in upload | No -- fix request |
| 400 | `invalidMetadata` | Invalid video metadata | No -- fix request |
| 401 | `authError` | Token expired or invalid | Yes -- refresh token |
| 403 | `forbidden` | Insufficient permissions | No -- check scopes |
| 403 | `quotaExceeded` | Daily quota exhausted | No -- wait until midnight PT |
| 403 | `rateLimitExceeded` | Too many requests per second | Yes -- exponential backoff |
| 403 | `commentsDisabled` | Comments disabled on video | No |
| 404 | `videoNotFound` | Video does not exist or is private | No |
| 404 | `playlistNotFound` | Playlist does not exist | No |
| 409 | `conflict` | Resource version conflict | Yes -- re-fetch and retry |
| 500 | `internalError` | YouTube server error | Yes -- exponential backoff |
| 503 | `backendError` | YouTube temporarily unavailable | Yes -- exponential backoff |

**Error Response Format:**
```json
{
  "error": {
    "code": 403,
    "message": "The request cannot be completed because you have exceeded your quota.",
    "errors": [
      {
        "message": "The request cannot be completed because you have exceeded your quota.",
        "domain": "youtube.quota",
        "reason": "quotaExceeded"
      }
    ]
  }
}
```

### 4.8 Mobile-Specific Notes

- **Video Upload from Device:** Always use the resumable upload protocol for videos from mobile. Chunk size recommendation: 5 MB per chunk on cellular, 10 MB on Wi-Fi. Use `react-native-background-upload` for uploads that continue when the app is backgrounded.
- **Video Compression:** Compress to H.264 MP4 at 720p (1280x720), 30fps, 2-5 Mbps bitrate for standard uploads. For higher quality, use 1080p at 8 Mbps. YouTube re-encodes all uploads, so extremely high bitrates offer diminishing returns.
- **Thumbnail Generation:** Auto-generate a thumbnail from the video before upload using `react-native-video-processing`. Show a thumbnail selector with 3-5 auto-extracted frames. Allow uploading a custom thumbnail if the channel is verified.
- **Quota Dashboard:** Display a quota usage indicator in the Esmer UI showing remaining daily quota. Warn users when quota is below 20% with suggestions to postpone non-critical operations.
- **Content Preview:** Show a preview card with video title, description preview, privacy status badge, scheduled publish time, selected thumbnail, and estimated quota cost before initiating the upload.
- **Upload Progress:** Show a multi-stage progress indicator: "Compressing" (local), "Uploading" (chunk progress), "Processing" (YouTube processing), "Live" (published).
- **Offline Metadata Editing:** Allow editing video titles, descriptions, and tags offline, then sync changes when connectivity returns (50 quota units per update).
- **Analytics Charts:** Render sparkline charts for views, watch time, and subscriber growth. Cache analytics data locally since YouTube data has a 24-48 hour delay anyway.

---

## 5. Reddit (Reddit API)

### 5.1 Service Overview

Reddit is the largest discussion platform organized into topic-based communities (subreddits). Esmer integrates with the Reddit API to enable users to submit posts (text, link, image, video) to subreddits, read and search posts, manage comments and replies, monitor mentions, view subreddit information, and track post performance. Reddit's API uses OAuth2 exclusively and requires a custom User-Agent header on all requests.

### 5.2 Authentication

**Method:** OAuth 2.0 (Authorization Code Flow)

| Parameter | Value |
|---|---|
| Authorization URL | `https://www.reddit.com/api/v1/authorize` |
| Token URL | `https://www.reddit.com/api/v1/access_token` |
| Revoke URL | `https://www.reddit.com/api/v1/revoke_token` |
| Grant Type | `authorization_code` |
| Token Lifetime | 1 hour (access token), permanent (refresh token) |

**Note:** Token requests use HTTP Basic Auth with `client_id:client_secret` in the `Authorization` header, and `Content-Type: application/x-www-form-urlencoded`.

**Required Scopes:**

| Scope | Purpose |
|---|---|
| `identity` | Read the user's Reddit username and account info |
| `read` | Read posts, comments, subreddit info |
| `submit` | Submit posts and comments |
| `edit` | Edit own posts and comments |
| `history` | Read user's post and comment history |
| `mysubreddits` | Read subscribed subreddits |
| `subscribe` | Subscribe/unsubscribe to subreddits |
| `vote` | Upvote/downvote posts and comments |
| `save` | Save/unsave posts and comments |
| `report` | Report posts and comments |
| `privatemessages` | Read and send private messages |
| `flair` | Set flair on posts |
| `modflair` | Manage flair as a moderator |
| `modposts` | Approve, remove, spam posts/comments |
| `modconfig` | Manage subreddit configuration |
| `modlog` | Read moderation log |
| `modmail` | Read and send modmail |

**Recommendations for Esmer:**
- Request `identity`, `read`, `submit`, `edit`, `vote`, `save`, `history`, and `mysubreddits` for standard users.
- Add `modposts`, `modflair`, `modlog`, and `modmail` for users who moderate subreddits.
- Access tokens expire in 1 hour. Use refresh tokens to obtain new access tokens.
- Use `duration=permanent` in the authorization URL to get a refresh token.

**Mandatory User-Agent:**
Reddit requires a descriptive User-Agent header on all API requests:
```
User-Agent: esmer:v1.0.0 (by /u/esmer_official)
```
Requests without a proper User-Agent are rate-limited or blocked.

### 5.3 Base URL

```
https://oauth.reddit.com
```

Note: Authenticated requests use `oauth.reddit.com`, not `www.reddit.com`.

### 5.4 Rate Limits

| Limit Type | Value |
|---|---|
| Requests per minute (OAuth) | 100 requests/minute per OAuth client |
| Requests per minute (non-OAuth) | 10 requests/minute per IP |
| Listing page size | Max 100 items per request |
| Search result limit | Max 250 results per query |
| Post submission cooldown | Varies by karma and subreddit (2-10 minutes for new accounts) |
| Comment submission cooldown | Varies by karma (may be immediate for established accounts) |
| Concurrent connections | 2 per client |

**Rate Limit Headers:**
- `X-Ratelimit-Used` -- Number of requests made in the current window
- `X-Ratelimit-Remaining` -- Remaining requests in the current window
- `X-Ratelimit-Reset` -- Seconds until the window resets

**Retry Strategy:** On `429 Too Many Requests`, wait `X-Ratelimit-Reset` seconds. Maintain a request queue with minimum 600ms spacing between requests.

### 5.5 API Endpoints

#### Posts

**Submit a Text Post**
```
POST /api/submit
```
- **Content-Type:** `application/x-www-form-urlencoded`
- **Parameters:**
  - `kind=self` (text post)
  - `sr` -- Subreddit name (without `r/` prefix)
  - `title` -- Post title (max 300 characters)
  - `text` -- Post body in Markdown (max 40,000 characters)
  - `flair_id` -- Post flair ID (optional)
  - `flair_text` -- Post flair text (optional)
  - `nsfw` -- `true` or `false`
  - `spoiler` -- `true` or `false`
  - `sendreplies` -- `true` to receive reply notifications
  - `resubmit` -- `true` to allow resubmission of duplicate links
- **Response (200 OK):**
```json
{
  "json": {
    "errors": [],
    "data": {
      "url": "https://www.reddit.com/r/esmer/comments/abc123/my_post_title/",
      "drafts_count": 0,
      "id": "abc123",
      "name": "t3_abc123"
    }
  }
}
```

**Submit a Link Post**
```
POST /api/submit
```
- **Parameters:**
  - `kind=link`
  - `sr` -- Subreddit name
  - `title` -- Post title
  - `url` -- Link URL
  - Other parameters same as text post
- **Response:** Same structure as text post.

**Submit an Image Post**
```
POST /api/submit
```
- **Parameters:**
  - `kind=image`
  - `sr` -- Subreddit name
  - `title` -- Post title
  - `url` -- Direct image URL (hosted on Reddit's media servers)
- **Note:** Image must first be uploaded via the media upload endpoint (see below).

**Upload Media (for image/video posts)**
```
POST /api/media/asset.json
```
- **Request Body (JSON):**
```json
{
  "filepath": "image.jpg",
  "mimetype": "image/jpeg"
}
```
- **Response:**
```json
{
  "args": {
    "action": "https://reddit-uploaded-media.s3-accelerate.amazonaws.com",
    "fields": [
      { "name": "key", "value": "..." },
      { "name": "AWSAccessKeyId", "value": "..." },
      { "name": "policy", "value": "..." },
      { "name": "signature", "value": "..." },
      { "name": "Content-Type", "value": "image/jpeg" },
      { "name": "x-amz-security-token", "value": "..." }
    ]
  },
  "asset": {
    "asset_id": "...",
    "processing_state": "incomplete",
    "payload": { "filepath": "..." },
    "websocket_url": "wss://..."
  }
}
```
- Then POST the file as multipart/form-data to the S3 URL with the provided fields.

**Get a Post**
```
GET /api/info
```
- **Query Parameters:**
  - `id` -- Fullname (e.g., `t3_abc123`), or comma-separated list
- **Alternative:**
```
GET /r/{subreddit}/comments/{post_id}
```
- **Query Parameters:**
  - `sort` -- `confidence`, `top`, `new`, `controversial`, `old`, `qa`
  - `limit` -- Number of comments to return
  - `depth` -- Max comment depth
- **Response:** Array of two Listing objects: [0] is the post, [1] is the comment tree.

**Get Posts from a Subreddit**
```
GET /r/{subreddit}/{sort}
```
- **Path Parameters:**
  - `subreddit` -- Subreddit name
  - `sort` -- `hot`, `new`, `rising`, `controversial`, `top`
- **Query Parameters:**
  - `limit` -- 1-100 (default 25)
  - `after` -- Fullname for forward pagination
  - `before` -- Fullname for backward pagination
  - `t` -- Time range for `top`/`controversial`: `hour`, `day`, `week`, `month`, `year`, `all`
  - `count` -- Number of items already seen
- **Response (200 OK):**
```json
{
  "kind": "Listing",
  "data": {
    "after": "t3_xyz789",
    "before": null,
    "children": [
      {
        "kind": "t3",
        "data": {
          "id": "abc123",
          "name": "t3_abc123",
          "title": "Post Title",
          "selftext": "Post body text in Markdown",
          "selftext_html": "<p>Post body text in HTML</p>",
          "author": "username",
          "subreddit": "esmer",
          "score": 150,
          "upvote_ratio": 0.92,
          "num_comments": 42,
          "created_utc": 1700000000,
          "permalink": "/r/esmer/comments/abc123/post_title/",
          "url": "https://www.reddit.com/r/esmer/comments/abc123/post_title/",
          "thumbnail": "https://b.thumbs.redditmedia.com/...",
          "preview": {
            "images": [
              {
                "source": { "url": "https://preview.redd.it/...", "width": 1200, "height": 675 },
                "resolutions": [
                  { "url": "...", "width": 108, "height": 60 },
                  { "url": "...", "width": 216, "height": 121 },
                  { "url": "...", "width": 320, "height": 180 }
                ]
              }
            ]
          },
          "is_self": true,
          "over_18": false,
          "spoiler": false,
          "stickied": false,
          "locked": false,
          "archived": false
        }
      }
    ],
    "dist": 25
  }
}
```

**Search Posts**
```
GET /r/{subreddit}/search
```
- **Query Parameters:**
  - `q` -- Search query (supports `site:`, `author:`, `selftext:`, `title:`, `subreddit:`, `url:`, `flair:`)
  - `restrict_sr` -- `true` to limit to subreddit, `false` for all of Reddit
  - `sort` -- `relevance`, `hot`, `top`, `new`, `comments`
  - `t` -- Time range: `hour`, `day`, `week`, `month`, `year`, `all`
  - `limit` -- 1-100
  - `after`, `before` -- Pagination
  - `type` -- `link`, `sr` (subreddit), `user`
- **Response:** Listing of search results.

**Search All of Reddit**
```
GET /search
```
- Same parameters as subreddit search, without `restrict_sr`.

**Delete a Post**
```
POST /api/del
```
- **Content-Type:** `application/x-www-form-urlencoded`
- **Parameters:**
  - `id` -- Fullname of the post (e.g., `t3_abc123`)
- **Response:** Empty JSON `{}` on success.

**Edit a Post**
```
POST /api/editusertext
```
- **Parameters:**
  - `thing_id` -- Fullname (e.g., `t3_abc123`)
  - `text` -- New Markdown text
- **Response:** Updated post data.

#### Comments

**Create a Comment**
```
POST /api/comment
```
- **Content-Type:** `application/x-www-form-urlencoded`
- **Parameters:**
  - `parent` -- Fullname of the post or comment being replied to (e.g., `t3_abc123` or `t1_def456`)
  - `text` -- Comment body in Markdown (max 10,000 characters)
- **Response (200 OK):**
```json
{
  "json": {
    "errors": [],
    "data": {
      "things": [
        {
          "kind": "t1",
          "data": {
            "id": "ghi789",
            "name": "t1_ghi789",
            "body": "Comment text here",
            "author": "username",
            "parent_id": "t3_abc123",
            "created_utc": 1700000000
          }
        }
      ]
    }
  }
}
```

**Get Comments for a Post**
```
GET /r/{subreddit}/comments/{post_id}
```
- **Query Parameters:**
  - `sort` -- `confidence` (best), `top`, `new`, `controversial`, `old`, `qa`
  - `limit` -- Max comments to return
  - `depth` -- Max nesting depth
  - `comment` -- Specific comment ID to focus on
  - `context` -- Number of parent comments to include for focused comment
- **Response:** Two-element array: post listing + comment tree listing.

**Edit a Comment**
```
POST /api/editusertext
```
- **Parameters:**
  - `thing_id` -- `t1_{commentId}`
  - `text` -- Updated Markdown text

**Delete a Comment**
```
POST /api/del
```
- **Parameters:**
  - `id` -- `t1_{commentId}`

#### Voting

**Vote on a Post or Comment**
```
POST /api/vote
```
- **Content-Type:** `application/x-www-form-urlencoded`
- **Parameters:**
  - `id` -- Fullname (e.g., `t3_abc123` or `t1_def456`)
  - `dir` -- `1` (upvote), `-1` (downvote), `0` (remove vote)
- **Response:** Empty `{}` on success.

#### Save/Unsave

**Save a Post or Comment**
```
POST /api/save
```
- **Parameters:**
  - `id` -- Fullname
  - `category` -- Save category (Reddit Premium only)

**Unsave**
```
POST /api/unsave
```
- **Parameters:**
  - `id` -- Fullname

#### Users and Profiles

**Get Authenticated User**
```
GET /api/v1/me
```
- **Response (200 OK):**
```json
{
  "id": "user_id",
  "name": "username",
  "created_utc": 1500000000,
  "link_karma": 5000,
  "comment_karma": 15000,
  "total_karma": 20000,
  "icon_img": "https://styles.redditmedia.com/...",
  "is_gold": false,
  "has_verified_email": true,
  "num_friends": 25,
  "subreddit": {
    "display_name": "u_username",
    "subscribers": 100,
    "title": "User's profile title",
    "public_description": "Bio text"
  }
}
```

**Get User by Username**
```
GET /user/{username}/about
```
- **Response:** User object with `name`, `link_karma`, `comment_karma`, `created_utc`, `icon_img`, etc.

**Get User's Posts**
```
GET /user/{username}/submitted
```
- **Query Parameters:** `sort`, `t`, `limit`, `after`, `before`
- **Response:** Listing of the user's submitted posts.

**Get User's Comments**
```
GET /user/{username}/comments
```
- Same pagination parameters as submitted posts.

#### Subreddits

**Get Subreddit Info**
```
GET /r/{subreddit}/about
```
- **Response (200 OK):**
```json
{
  "kind": "t5",
  "data": {
    "id": "subreddit_id",
    "display_name": "esmer",
    "title": "Esmer Community",
    "public_description": "Official community for Esmer AI assistant",
    "description": "Full sidebar description in Markdown",
    "subscribers": 50000,
    "active_user_count": 250,
    "created_utc": 1600000000,
    "over18": false,
    "icon_img": "https://styles.redditmedia.com/...",
    "banner_background_image": "https://styles.redditmedia.com/...",
    "submit_text": "Submission guidelines...",
    "submission_type": "any",
    "allow_images": true,
    "allow_videos": true
  }
}
```

**Search Subreddits**
```
GET /subreddits/search
```
- **Query Parameters:**
  - `q` -- Search query
  - `limit` -- 1-100
  - `after`, `before` -- Pagination

**Get User's Subscribed Subreddits**
```
GET /subreddits/mine/subscriber
```
- **Query Parameters:** `limit`, `after`, `before`
- **Response:** Listing of subscribed subreddits.

**Get Popular Subreddits**
```
GET /subreddits/popular
```

**Get New Subreddits**
```
GET /subreddits/new
```

#### Private Messages

**Get Inbox**
```
GET /message/inbox
```
- **Query Parameters:** `limit`, `after`, `before`, `mark=true` (mark as read)
- **Response:** Listing of messages.

**Send a Private Message**
```
POST /api/compose
```
- **Parameters:**
  - `to` -- Recipient username
  - `subject` -- Message subject (max 100 characters)
  - `text` -- Message body in Markdown

**Mark Messages as Read**
```
POST /api/read_message
```
- **Parameters:**
  - `id` -- Comma-separated fullnames of messages

### 5.6 Webhooks / Real-time

Reddit does not provide a webhook API. Options for real-time monitoring:

**Polling:** The only supported method. Poll endpoints at regular intervals.
- For mentions: `GET /message/inbox` (check for username mentions)
- For post replies: `GET /message/inbox?mark=false` (check for comment replies)
- For subreddit new posts: `GET /r/{subreddit}/new` with `limit=10`

**Reddit Streaming (unofficial):** Reddit offers a streaming endpoint for comments:
```
GET https://www.reddit.com/r/{subreddit}/comments.json?limit=100
```
Poll every 2-5 seconds with `before` parameter to get only new comments. This is not an official streaming API but is tolerated within rate limits.

**Esmer Strategy:**
- Poll user inbox every 60 seconds for mentions and replies.
- For monitored subreddits, poll `/new` every 30-60 seconds (within the 100 req/min limit).
- Use the `before` parameter and track the last-seen fullname to avoid processing duplicates.
- For post performance monitoring, poll post details every 5 minutes and track score/comment changes.

### 5.7 Error Handling

| HTTP Code | Description | Retry? |
|---|---|---|
| 200 | Success (check `json.errors` array for logical errors) | N/A |
| 301 | Subreddit name redirect (case sensitivity) | Yes -- follow redirect |
| 302 | Authentication redirect (token expired) | Yes -- refresh token |
| 400 | Bad request (malformed parameters) | No -- fix request |
| 401 | Unauthorized (token expired or invalid) | Yes -- refresh token |
| 403 | Forbidden (banned from subreddit, insufficient permissions) | No |
| 404 | Subreddit or resource not found | No |
| 409 | Conflict (e.g., duplicate submission) | No |
| 429 | Rate limit exceeded | Yes -- wait per `X-Ratelimit-Reset` |
| 500 | Reddit server error | Yes -- exponential backoff |
| 502 | Bad gateway | Yes -- exponential backoff |
| 503 | Service unavailable | Yes -- exponential backoff |

**Note:** Reddit often returns `200 OK` with errors in the response body:
```json
{
  "json": {
    "errors": [
      ["RATELIMIT", "you are doing that too much. try again in 7 minutes.", "ratelimit"]
    ]
  }
}
```
Always parse the `json.errors` array even on `200 OK` responses.

**Error Response Format (non-200):**
```json
{
  "message": "Forbidden",
  "error": 403,
  "reason": "banned"
}
```

### 5.8 Mobile-Specific Notes

- **Image Upload from Device:** Reddit requires uploading images via the S3 pre-signed URL flow (`/api/media/asset.json`). Compress images to JPEG quality 85, max 20 MB. Use the WebSocket URL returned by the upload endpoint to monitor upload processing status.
- **Video Upload:** Reddit supports video uploads up to 1 GB and 15 minutes. Use the same media asset upload flow. Compress to H.264 MP4 at 720p.
- **Markdown Editor:** Provide a mobile-friendly Markdown editor with formatting toolbar (bold, italic, links, lists, code blocks, quotes). Reddit uses a specific Markdown flavor -- preview the rendered output before submission.
- **Post Type Selector:** Present a clear post type picker (text, link, image, video, poll) since each requires different parameters and upload flows.
- **Subreddit Rules:** Before submission, fetch `GET /r/{subreddit}/about/rules` and display the rules to the user. Some subreddits require specific flair or have title formatting requirements.
- **Karma Requirements:** Some subreddits have minimum karma requirements for posting. If a submission fails with a `RATELIMIT` or permissions error, inform the user about potential karma thresholds.
- **Comment Trees:** Reddit comment threads can be deeply nested. Implement collapsible comment trees with a "load more" mechanism for threads with many replies. Limit initial render depth to 3-4 levels on mobile.
- **Content Preview:** Show a Reddit-style card preview with the post title, subreddit icon, post type indicator (text/link/image), and a character/word count.
- **Offline Drafts:** Queue posts and comments locally when offline. Warn users that Reddit has anti-spam measures that may reject submissions if too many are sent in rapid succession upon reconnection.

---

## 6. Medium (Medium API)

### 6.1 Service Overview

Medium is a long-form content publishing platform. Esmer integrates with the Medium API to enable users to publish articles, manage publications, and cross-post content. **Important caveat:** Medium has officially deprecated and stopped supporting its API as of 2023. No new API keys or OAuth applications can be created. Existing integrations with previously issued tokens may still function. Esmer should treat this integration as legacy/maintenance-only and consider unofficial alternatives (such as using Medium's RSS feeds or third-party services).

### 6.2 Authentication

**Method:** OAuth 2.0 or Self-Issued Access Token (both deprecated)

| Parameter | Value |
|---|---|
| Authorization URL | `https://medium.com/m/oauth/authorize` |
| Token URL | `https://api.medium.com/v1/tokens` |
| Grant Type | `authorization_code` |

**Self-Issued Access Token:**
- Generated from Medium Settings > Security and Apps > Integration Tokens
- No expiration (valid until manually revoked)
- No new tokens can be created (API deprecated)

**OAuth2 Scopes:**

| Scope | Purpose |
|---|---|
| `basicProfile` | Read user's profile (id, username, name, url, imageUrl) |
| `publishPost` | Create posts on behalf of the user |
| `listPublications` | List publications the user is a part of |

**Recommendations for Esmer:**
- Only support existing tokens from users who previously created them.
- Display a deprecation notice in the Esmer UI indicating Medium API is no longer actively supported.
- For new users, suggest using RSS-to-Medium republishing tools or manual posting.
- Do not invest in new feature development for the Medium integration.

### 6.3 Base URL

```
https://api.medium.com/v1
```

### 6.4 Rate Limits

Medium's documented rate limits (when the API was active):

| Limit Type | Value |
|---|---|
| Requests per second | Not explicitly documented |
| Posts per day | Estimated ~10 posts/day per user (anti-spam) |
| API availability | Deprecated -- may be intermittently unavailable |

**Throttle Response:** Standard `429 Too Many Requests`. Given the deprecated status, rate limits may be more aggressive or endpoints may return `503`.

### 6.5 API Endpoints

#### User

**Get Authenticated User**
```
GET /v1/me
```
- **Headers:** `Authorization: Bearer {access_token}`, `Content-Type: application/json`, `Accept: application/json`
- **Response (200 OK):**
```json
{
  "data": {
    "id": "5303d74c64f66366f00cb9b2a94f3251bf5",
    "username": "esmeruser",
    "name": "Esmer User",
    "url": "https://medium.com/@esmeruser",
    "imageUrl": "https://cdn-images-1.medium.com/fit/c/200/200/..."
  }
}
```

#### Publications

**Get User's Publications**
```
GET /v1/users/{userId}/publications
```
- **Response (200 OK):**
```json
{
  "data": [
    {
      "id": "b969ac62a46b",
      "name": "Esmer Blog",
      "description": "The official Esmer publication",
      "url": "https://medium.com/esmer-blog",
      "imageUrl": "https://cdn-images-1.medium.com/..."
    }
  ]
}
```

**Get Publication Contributors**
```
GET /v1/publications/{publicationId}/contributors
```
- **Response (200 OK):**
```json
{
  "data": [
    {
      "publicationId": "b969ac62a46b",
      "userId": "5303d74c64f66366f00cb9b2a94f3251bf5",
      "role": "editor"
    }
  ]
}
```

#### Posts

**Create a Post (User)**
```
POST /v1/users/{userId}/posts
```
- **Request Body:**
```json
{
  "title": "How Esmer Delegates Tasks with AI",
  "contentFormat": "html",
  "content": "<h1>How Esmer Delegates Tasks</h1><p>Esmer is an AI-powered delegation assistant that connects to cloud APIs on your behalf...</p>",
  "tags": ["ai", "productivity", "automation"],
  "canonicalUrl": "https://esmer.app/blog/ai-delegation",
  "publishStatus": "public",
  "license": "all-rights-reserved",
  "notifyFollowers": true
}
```
- **Parameters:**
  - `title` (required) -- Post title
  - `contentFormat` (required) -- `html` or `markdown`
  - `content` (required) -- Post body in the specified format
  - `tags` -- Array of up to 5 tags
  - `canonicalUrl` -- Original URL if cross-posting
  - `publishStatus` -- `public`, `draft`, `unlisted`
  - `license` -- `all-rights-reserved`, `cc-40-by`, `cc-40-by-sa`, `cc-40-by-nd`, `cc-40-by-nc`, `cc-40-by-nc-nd`, `cc-40-by-nc-sa`, `cc-40-zero`, `public-domain`
  - `notifyFollowers` -- `true` to notify followers
- **Response (201 Created):**
```json
{
  "data": {
    "id": "e6f36a",
    "title": "How Esmer Delegates Tasks with AI",
    "authorId": "5303d74c64f66366f00cb9b2a94f3251bf5",
    "tags": ["ai", "productivity", "automation"],
    "url": "https://medium.com/@esmeruser/how-esmer-delegates-tasks-with-ai-e6f36a",
    "canonicalUrl": "https://esmer.app/blog/ai-delegation",
    "publishStatus": "public",
    "publishedAt": 1700000000000,
    "license": "all-rights-reserved",
    "licenseUrl": "https://policy.medium.com/medium-terms-of-service-9db0094a1e0f"
  }
}
```

**Create a Post under a Publication**
```
POST /v1/publications/{publicationId}/posts
```
- Same request body as user posts.
- The authenticated user must be a contributor or editor of the publication.
- **Response:** Same structure as user post, with `publicationId` field included.

**Content Format Notes:**
- **HTML:** Supports `<h1>`-`<h4>`, `<p>`, `<a>`, `<blockquote>`, `<code>`, `<pre>`, `<ul>`, `<ol>`, `<li>`, `<strong>`, `<em>`, `<img>`, `<figure>`, `<figcaption>`
- **Markdown:** Standard Markdown with support for images via `![alt](url)`
- **Images in content:** Must be hosted externally (URL references). Medium will download and re-host them. Supported formats: JPEG, PNG, GIF. Max recommended: 25 MB per image.

### 6.6 Webhooks / Real-time

Medium does not provide any webhook or real-time API.

**Alternatives for Monitoring:**
- **RSS Feeds:** Every Medium user and publication has an RSS feed:
  - User feed: `https://medium.com/feed/@{username}`
  - Publication feed: `https://medium.com/feed/{publication-slug}`
  - Tag feed: `https://medium.com/feed/tag/{tag}`
- Poll RSS feeds every 15-30 minutes to detect new publications.
- Parse the RSS XML to extract post titles, URLs, publication dates, and content previews.

### 6.7 Error Handling

| HTTP Code | Description | Retry? |
|---|---|---|
| 200 | Success | N/A |
| 201 | Created (post published) | N/A |
| 400 | Bad request (malformed JSON, missing required fields) | No -- fix request |
| 401 | Unauthorized (invalid or expired token) | No -- API deprecated, cannot refresh |
| 403 | Forbidden (not a contributor of the publication) | No |
| 429 | Rate limit exceeded | Yes -- wait and retry |
| 500 | Medium server error | Yes -- exponential backoff |

**Error Response Format:**
```json
{
  "errors": [
    {
      "message": "Token was invalid.",
      "code": 6003
    }
  ]
}
```

**Error Codes:**

| Code | Description |
|---|---|
| 6000 | General server error |
| 6001 | Required field missing |
| 6002 | Invalid field value |
| 6003 | Invalid or expired token |
| 6004 | Content format invalid |
| 6005 | Publication not found |
| 6006 | User not authorized for publication |

### 6.8 Mobile-Specific Notes

- **Deprecation Warning:** Display a prominent banner in Esmer indicating that Medium API integration is deprecated and may stop working at any time without notice.
- **Content Composition:** Provide a rich text editor for long-form content. Convert the editor output to HTML before sending to Medium's API. Support inline images by first uploading them to Esmer's own storage (or a CDN) and embedding the URLs in the HTML.
- **Cross-posting Workflow:** The primary use case for Medium via Esmer is cross-posting content originally authored for another platform (blog, LinkedIn article). Set the `canonicalUrl` to the original post's URL to avoid SEO duplication penalties.
- **Draft vs Publish:** Default to `publishStatus: "draft"` to let users review on Medium before publishing. Provide a toggle for direct publishing.
- **Tags:** Medium allows up to 5 tags per post. Suggest relevant tags based on the post content using Esmer's AI capabilities.
- **Fallback Strategy:** If the Medium API becomes fully unavailable, implement a "copy to clipboard with Medium formatting" feature that helps users manually paste content into Medium's web editor.

---

## Cross-Service Comparison Matrix

### Authentication Methods

| Service | OAuth 2.0 | OAuth 1.0a | API Key | App Token |
|---|---|---|---|---|
| X / Twitter | Yes (PKCE) | Deprecated | No | Yes (App-Only Bearer) |
| LinkedIn | Yes | No | No | No |
| Facebook Graph API | Yes | No | No | Yes (App Access Token) |
| YouTube | Yes (Google) | No | Yes (read-only) | No |
| Reddit | Yes | No | No | No |
| Medium | Yes (deprecated) | No | Yes (deprecated) | No |

### Content Publishing Capabilities

| Feature | X/Twitter | LinkedIn | Facebook | YouTube | Reddit | Medium |
|---|---|---|---|---|---|---|
| Text posts | 280/25K chars | 3,000 chars | 63,206 chars | N/A (video-only) | 40,000 chars | Unlimited (articles) |
| Image posts | Up to 4 images | Single image | Multiple images | N/A | Single/gallery | Inline in articles |
| Video posts | Up to 140s | Up to 15 min | Up to 4 hours | Up to 12 hours | Up to 15 min | Not supported |
| Link sharing | Yes (t.co wrapped) | Yes (article card) | Yes (preview card) | N/A | Yes (link posts) | Yes (canonical URL) |
| Polls | Yes (2-4 options) | No | No | Community polls | Yes (prediction) | No |
| Scheduled posts | No (via Esmer queue) | No (via Esmer queue) | Yes (native) | Yes (native) | No (via Esmer queue) | No |
| Stories/Shorts | No | No | Stories via separate API | Shorts (as videos) | No | No |

### Rate Limits Summary

| Service | Primary Limit | Window | Throttle Response |
|---|---|---|---|
| X / Twitter (Basic) | 100 tweets/24h per user | 24 hours | 429 + x-rate-limit-reset |
| LinkedIn | 150 posts/day per member | 24 hours | 429 + Retry-After |
| Facebook | 250 posts/24h per Page | 24 hours | 429 or error code 4/32 |
| YouTube | 10,000 quota units/day | 24 hours (Pacific) | 403 quotaExceeded |
| Reddit | 100 req/min per client | 1 minute | 429 + X-Ratelimit-Reset |
| Medium | ~10 posts/day (est.) | 24 hours | 429 |

### Webhook / Real-time Capabilities

| Service | Native Webhooks | Streaming API | Recommended Polling Interval |
|---|---|---|---|
| X / Twitter | Enterprise only (AAAPI) | Yes (Filtered Stream, Pro+) | 60s (Basic tier) |
| LinkedIn | No | No | 5-15 minutes |
| Facebook | Yes (Webhooks product) | No | Not needed (use webhooks) |
| YouTube | PubSubHubbub (new videos only) | No | 5-10 minutes (comments) |
| Reddit | No | No (unofficial poll-based) | 30-60 seconds |
| Medium | No | No | 15-30 minutes (RSS) |

### Media Upload Constraints

| Service | Max Image Size | Max Video Size | Max Video Duration | Upload Method |
|---|---|---|---|---|
| X / Twitter | 5 MB | 512 MB | 140 seconds | Chunked (v1.1) |
| LinkedIn | 10 MB | 200 MB | 15 minutes | Two-step (init + PUT) |
| Facebook | 10 MB | 10 GB | 4 hours | Resumable (multi-phase) |
| YouTube | N/A | 256 GB | 12 hours | Resumable |
| Reddit | 20 MB | 1 GB | 15 minutes | S3 pre-signed URL |
| Medium | 25 MB (inline) | N/A | N/A | URL reference in content |

### Mobile Integration Priority

| Service | Priority | Rationale |
|---|---|---|
| X / Twitter | P0 (Critical) | Highest-frequency social posting; real-time engagement |
| Facebook | P0 (Critical) | Largest social platform; Page management is core use case |
| LinkedIn | P1 (High) | Professional audience; organization posting for B2B users |
| YouTube | P1 (High) | Video management; high value but lower posting frequency |
| Reddit | P2 (Medium) | Community engagement; niche but highly engaged user base |
| Medium | P3 (Low) | Deprecated API; legacy support only |

### API Maturity and Stability

| Service | API Version | Stability | Breaking Change Risk |
|---|---|---|---|
| X / Twitter | v2 (active), v1.1 (media only) | Moderate -- frequent tier/pricing changes | High (pricing model shifts) |
| LinkedIn | REST, versioned (202404) | High -- Microsoft backing, versioned | Low (explicit versioning) |
| Facebook | Graph API v21.0 | High -- regular versioned releases | Low (2-year deprecation cycle) |
| YouTube | Data API v3 | Very High -- stable since 2014 | Very Low |
| Reddit | Unversioned | Moderate -- recent API access changes | Medium (access policy changes) |
| Medium | v1 (deprecated) | Very Low -- officially unsupported | Critical (may disappear) |

---

## Appendix A: Recommended Esmer Implementation Patterns

### Unified Post Interface

Esmer should expose a unified post interface across all social media providers:

```
EsmerSocialPost {
  id: string                    // Provider-specific post ID
  provider: string              // "twitter" | "linkedin" | "facebook" | "youtube" | "reddit" | "medium"
  type: string                  // "text" | "image" | "video" | "link" | "article" | "poll"
  author: {
    id: string                  // Provider-specific user/page/channel ID
    name: string
    handle: string              // @username, page name, channel name
    avatarUrl: string
    isOrganization: boolean     // true for pages, orgs, channels
  }
  content: {
    text: string                // Post text / caption / title
    html: string                // HTML content (Medium articles)
    markdown: string            // Markdown content (Reddit)
    links: string[]             // Attached URLs
    mediaIds: string[]          // Provider-specific media IDs
  }
  metrics: {
    likes: number               // Likes / upvotes / reactions
    shares: number              // Retweets / reposts / shares
    comments: number            // Replies / comments
    views: number               // Impressions / views
    engagementRate: number      // Calculated engagement percentage
  }
  status: string                // "draft" | "scheduled" | "published" | "deleted"
  scheduledAt: ISO8601 | null   // Scheduled publish time
  publishedAt: ISO8601 | null   // Actual publish time
  permalink: string             // Direct URL to the post on the platform
  platformConstraints: {
    maxTextLength: number
    maxImages: number
    maxVideoSize: number
    maxVideoDuration: number
    supportsPolls: boolean
    supportsScheduling: boolean
  }
}
```

### Cross-Posting Strategy

```
function crossPost(content, platforms):
    for platform in platforms:
        adapted = adaptContent(content, platform.constraints)
        if adapted.requiresMediaUpload:
            mediaIds = uploadMedia(adapted.media, platform)
            adapted.mediaIds = mediaIds
        if platform.supportsScheduling and content.scheduledAt:
            result = schedulePost(adapted, platform)
        else:
            result = publishPost(adapted, platform)
        storeResult(result)

function adaptContent(content, constraints):
    if content.text.length > constraints.maxTextLength:
        content.text = truncate(content.text, constraints.maxTextLength - 3) + "..."
    if content.images.length > constraints.maxImages:
        content.images = content.images.slice(0, constraints.maxImages)
    if content.video and content.video.duration > constraints.maxVideoDuration:
        content.video = trimVideo(content.video, constraints.maxVideoDuration)
    return content
```

### Mobile Upload Queue

For unreliable mobile connections, implement a persistent upload queue:

1. User composes post with media in Esmer
2. Esmer compresses media locally (per platform constraints)
3. Post enters local queue (SQLite/Realm) with status `pending`
4. Background service processes queue:
   a. Upload media assets (resumable where supported)
   b. Create/schedule the post
   c. Update status to `published` or `failed`
5. On failure: increment retry count, apply exponential backoff
6. After 3 failures: notify user, move to `failed` status for manual retry
7. On connectivity change: re-evaluate queue and process pending items

### Token Refresh Orchestration

```
function ensureValidToken(provider, userId):
    token = getStoredToken(provider, userId)
    if token.expiresAt > now() + 5_MINUTES:
        return token  // Still valid with buffer
    if token.refreshToken:
        newToken = refreshToken(provider, token.refreshToken)
        storeToken(provider, userId, newToken)
        return newToken
    else:
        // Token cannot be refreshed (Medium, or revoked)
        notifyUser("Re-authorization required for " + provider)
        throw AuthExpiredError(provider)
```

### Engagement Monitoring Service

```
function monitorEngagement(userId):
    posts = getRecentPosts(userId, since=24_HOURS_AGO)
    for post in posts:
        currentMetrics = fetchMetrics(post.provider, post.id)
        previousMetrics = getCachedMetrics(post.id)
        if significantChange(currentMetrics, previousMetrics):
            notifyUser(userId, formatEngagementUpdate(post, currentMetrics, previousMetrics))
        cacheMetrics(post.id, currentMetrics)

function significantChange(current, previous):
    return (
        current.likes > previous.likes * 1.5 or
        current.comments > previous.comments + 10 or
        current.shares > previous.shares * 2 or
        current.views > previous.views * 2
    )
```

---

## Appendix B: Content Constraints Quick Reference

| Platform | Max Text | Hashtags | Mentions | Link Behavior | Formatting |
|---|---|---|---|---|---|
| X / Twitter | 280 (25K premium) | Auto-linked | @username | t.co wrapped (23 chars) | None |
| LinkedIn | 3,000 | Auto-linked | URN-based | Preview card | Line breaks only |
| Facebook | 63,206 | Auto-linked | @[name](id) tagging | Preview card | None in posts |
| YouTube | Title: 100, Desc: 5,000 | Auto-linked in desc. | N/A | Clickable in desc. | None |
| Reddit | Title: 300, Body: 40,000 | Not supported | /u/username | Link posts or inline | Markdown |
| Medium | Unlimited | Not tagged | N/A | Inline/embedded | Rich HTML/Markdown |

