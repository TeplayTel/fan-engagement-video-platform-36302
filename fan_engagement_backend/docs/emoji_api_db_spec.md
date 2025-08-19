# Emoji Backend and DB Spec Summary
Source: TEPlay Solution Design Document (Emoji) v1.0.1 (attachments/20250819_102954_TEPlay_Solution_Design_Document_Emoji_v1.0_1.pdf)

Scope
- Define DB schema for emoji assets and reactions (PostgreSQL).
- Define REST API endpoints for uploading emoji assets, listing emojis, capturing user reactions, and getting emoji statistics.

Database Schema (PostgreSQL)
1) Table: emoji_assets
- Purpose: Catalog of allowed reaction emojis with their canonical type and image location.
- Columns:
  - emoji_id SERIAL PRIMARY KEY
  - emoji_type VARCHAR(50) NOT NULL UNIQUE   // e.g., heart, clap, fire
  - emoji_path TEXT NOT NULL                 // storage path or absolute URL to image
  - created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP

Recommended indexes and constraints:
- UNIQUE (emoji_type)
- Consider CHECK (emoji_type ~ '^[a-z0-9_\\-]+$') to restrict to slugs
- If serving via CDN, treat emoji_path as image_url (absolute URL) or store relative path and prefix at runtime.

DDL:
CREATE TABLE IF NOT EXISTS emoji_assets (
  emoji_id SERIAL PRIMARY KEY,
  emoji_type VARCHAR(50) NOT NULL UNIQUE,
  emoji_path TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

2) Table: emoji_reactions
- Purpose: Record reactions sent by users for a given content “event” (e.g., video/live event).
- Columns:
  - reaction_id SERIAL PRIMARY KEY
  - event_id VARCHAR(100) NOT NULL          // content identifier, e.g., video_id or stream_id
  - user_id VARCHAR(100) NULL               // nullable if anonymous
  - emoji_id INT NOT NULL REFERENCES emoji_assets(emoji_id)
  - created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
  - updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP

Important note on source ambiguity:
- The PDF shows a line break artifact (“emoji_id,” followed by “emoji_type ... REFERENCES emoji_assets(emoji_type)”).
- To maintain normalization and remove ambiguity, this spec adopts a single FK: emoji_id → emoji_assets(emoji_id).
- The emoji_type can be derived via join when needed. If the business requires type-based FK, add a computed join; avoid duplicating both emoji_id and emoji_type in the reaction row.

Recommended indexes:
- INDEX (event_id)
- INDEX (emoji_id)
- Optional composite indexes for analytics:
  - INDEX (event_id, emoji_id)
  - INDEX (user_id, event_id)

DDL:
CREATE TABLE IF NOT EXISTS emoji_reactions (
  reaction_id SERIAL PRIMARY KEY,
  event_id VARCHAR(100) NOT NULL,
  user_id VARCHAR(100),
  emoji_id INT NOT NULL REFERENCES emoji_assets(emoji_id),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

Storage considerations
- The PDF mandates configuring a storage path and storing uploaded images. Options:
  - Local filesystem root (e.g., EMOJI_ASSETS_DIR=/var/app/assets/emojis) + serve via static files.
  - Object storage (S3/GCS) + store public URL in emoji_path.
- Ensure the API returns imageUrl for FE consumption. If emoji_path is relative, the BE should convert to absolute URL in responses.

API Endpoints (as per PDF)
Base path: /fan-engagement/emoji/v1

Authorization
- Upload requires admin token.
- List, Capture Reaction, and Stats allow admin or user tokens (user required for capture).
- Exact auth scheme is “Authorization: Bearer <token>” (RBAC not detailed in PDF).

1) Upload Emoji
- Method: POST
- URL: /fan-engagement/emoji/v1/upload
- Headers:
  - Content-Type: multipart/form-data
  - Authorization: Bearer your-admin-token
- Form fields:
  - emojiType: string (required) e.g., "heart"
  - emojiImage: file (required) binary file (e.g., heart.png)
- Behavior:
  - Validate emojiType uniqueness (case-insensitive recommended).
  - Store file to configured storage; create row in emoji_assets with emoji_type and emoji_path (image URL or path).
- Response (example):
  {
    "status": "SUCCESS",
    "message": "Emoji uploaded successfully",
    "data": {
      "emojiId": "EMJ103",        // Note: PDF uses string-like IDs in examples; our DB uses int. Map int to string if needed.
      "emojiType": "fire",
      "imageUrl": "https://cdn.ourdomain.com/emojis/fire.png"
    }
  }
- Errors:
  - 400: Missing fields, duplicate emojiType
  - 401/403: Unauthorized/Forbidden
  - 500: Storage or DB failure

2) Get Emoji List with Icons
- Method: GET
- URL: /fan-engagement/emoji/v1/listEmojis?pageNo=<int>&pageSize=<int>
- Headers:
  - Content-Type: application/json
  - Authorization: Bearer your-admin/user-token
- Behavior:
  - Fetch from emoji_assets, apply pagination (default pageNo=1, pageSize=50).
  - Sort by created_at ASC (or by emoji_type alphabetically) — PDF does not specify; choose consistent default.
- Response (example):
  {
    "status": "SUCCESS",
    "emojis": [
      {
        "emojiId": "EMJ103",
        "emojiType": "clap",
        "imageUrl": "https://cdn.mydomain.com/emojis/clap.png"
      },
      {
        "emojiId": "EMJ104",
        "emojiType": "fire",
        "imageUrl": "https://cdn.mydomain.com/emojis/fire.png"
      }
    ]
  }

3) Capture User Reaction
- Method: POST
- URL: /fan-engagement/emoji/v1/userEmojiReaction
- Headers:
  - Content-Type: application/json
  - Authorization: Bearer user-token
- Request body:
  {
    "userId": "USR456",       // optional per DB (nullable), but present in example
    "eventId": "EVT123",      // required
    "emojiId": "EMJ001",      // required; in DB, this is integer emoji_id
    "createdAt": "2025-07-29T11:35:24Z" // optional; server will set created_at if omitted
  }
- Behavior:
  - Validate emojiId exists.
  - Insert into emoji_reactions; ignore/override createdAt if server-side canonical time is preferred.
- Response (example):
  {
    "status": "SUCCESS",
    "message": "Reaction captured successfully",
    "data": { "reactionId": "R12345" }   // DB will return numeric id; format may be adapted
  }
- Errors:
  - 400: Missing or invalid fields
  - 401: Unauthorized
  - 404: emojiId not found

4) Get Emoji Statistics
- Method: GET
- URL: /fan-engagement/emoji/v1/stats?eventId=<id>&userId=<id>&pageNo=<int>&pageSize=<int>
- Headers:
  - Content-Type: application/json
  - Authorization: Bearer your-admin/user-token
- Behavior:
  - If eventId provided (common UI need): return aggregated counts per emoji for that event.
    SQL example:
      SELECT ea.emoji_id, ea.emoji_type, COUNT(*) AS count
      FROM emoji_reactions er
      JOIN emoji_assets ea ON ea.emoji_id = er.emoji_id
      WHERE er.event_id = $1
      GROUP BY ea.emoji_id, ea.emoji_type
      ORDER BY count DESC;
  - If userId provided: return per-event breakdown for that user (as per PDF example).
  - Pagination parameters present in URL; for aggregate lists with small cardinality (emoji set), pagination is typically unnecessary; for user event history, apply pageNo/pageSize.
- Responses (examples from PDF):
  Event-only:
  {
    "status": "SUCCESS",
    "eventId": "EVT123",
    "emojiStats": [
      {"emojiId": "EMJ103", "emojiType": "clap", "count": 45},
      {"emojiId": "EMJ105", "emojiType": "fire", "count": 32},
      {"emojiId": "EMJ104", "emojiType": "wow",  "count": 10}
    ]
  }

  User-only:
  {
    "status": "SUCCESS",
    "message": "User reaction stats fetched successfully",
    "data": {
      "userId": "USR456",
      "totalReactions": 4,
      "events": [
        {
          "eventId": "EVT1001",
          "eventType": "sports", // not stored in schema above; if needed, must be joined from content metadata service
          "emojiReactions": [
            { "emojiId": "EMJ101", "emojiType": "love", "count": 1 },
            { "emojiId": "EMJ105", "emojiType": "fire", "count": 1 }
          ]
        },
        ...
      ]
    }
  }

OpenAPI Shapes (concise)
- Upload Emoji (multipart/form-data):
  - 200: { status: "SUCCESS", message: string, data: { emojiId: string|int, emojiType: string, imageUrl: string } }
- List Emojis (application/json):
  - 200: { status: "SUCCESS", emojis: [{ emojiId: string|int, emojiType: string, imageUrl: string }] }
- Capture Reaction (application/json):
  - Request: { userId?: string, eventId: string, emojiId: string|int, createdAt?: string }
  - 200: { status: "SUCCESS", message: string, data: { reactionId: string|int } }
- Stats (application/json):
  - 200 (event): { status: "SUCCESS", eventId: string, emojiStats: [{ emojiId: string|int, emojiType: string, count: number }] }
  - 200 (user): { status: "SUCCESS", message: string, data: { userId: string, totalReactions: number, events: [{ eventId: string, eventType?: string, emojiReactions: [{ emojiId: string|int, emojiType: string, count: number }] }] } }

Validation & Business Rules
- emojiType must be unique, lowercase/slug recommended.
- On upload, reject unsupported file types (e.g., only .png/.svg/.webp) and size > configured limit.
- On reaction, only allow emojiId that exists; optionally rate limit per user (not covered in PDF).
- Timestamps: prefer server-side time; accept client-provided createdAt only if trusted.
- Event identifiers are opaque strings; eventType in examples implies integration with content metadata (out of scope of this spec).

Security
- OAuth/JWT bearer tokens implied. Admin-only for upload; users for list/stats and sending reactions.
- CORS allowed for FE origin(s).

Implementation Notes (FastAPI)
- Use python-multipart for upload handling (already included).
- Expose static files for local storage or return CDN URLs if using external storage.
- Return ID fields as strings in API to match PDF examples if desired; internally store as integers.

Response Field Naming
- The PDF uses camelCase in JSON. Adopt camelCase for external API even if Python uses snake_case internally.

Non-Goals / Out-of-scope (per PDF)
- WebSocket streaming of reactions is described in design notes but not specified in the PDF endpoints; can be considered future work.
- Event metadata (title, type) not provided by these endpoints.

Appendix: Example Mappings
- emojiId returned as "EMJ103" in examples can be constructed as prefix + zero-padded DB id (e.g., f"EMJ{emoji_id:03d}") if product wants this string format. Otherwise, return numeric IDs and let FE adapt.

