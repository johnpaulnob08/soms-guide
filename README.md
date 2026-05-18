# SOMS — Complete System Guide

## OSA-SACDEV Student Organization Management System

### Xavier University – Ateneo de Cagayan

> **Purpose of this guide:** Your complete defense reference. Explains every file, every API route, every button, every database collection, and how everything connects — written in plain language.

-----

## Table of Contents

1. [System Overview](#1-system-overview)
1. [File Structure](#2-file-structure)
1. [How the System Starts](#3-how-the-system-starts)
1. [Middleware — Code That Runs Before Every Request](#4-middleware)
1. [Firestore Database — Collections and What They Store](#5-firestore-database)
1. [All API Routes Explained](#6-all-api-routes-explained)
1. [Email Templates — When They Send and What They Say](#7-email-templates)
1. [Frontend Pages — Every Button and What It Does](#8-frontend-pages)
1. [How Backend Connects to Everything](#9-how-backend-connects-to-everything)
1. [CRUD Table](#10-crud-table)
1. [Security Model](#11-security-model)
1. [Design Decisions](#12-design-decisions)

-----

## 1. System Overview

SOMS is a **two-container Docker application**:

```
Browser (User)
    ↓  HTTP request to port 80
[Frontend Container]  ←→  Nginx web server
    ↓  /api/* requests forwarded internally
[Backend Container]   ←→  Node.js + Express.js on port 5000
    ↓  reads/writes data
[Firebase Firestore]  ←→  Cloud NoSQL database (Google)
    ↓  stores files
[Cloudinary CDN]      ←→  File and image storage
    ↓  sends emails
[Gmail SMTP]          ←→  Email delivery via Nodemailer
```

**Three types of users:**

|User                          |How They Log In                             |What They Can Do                      |
|------------------------------|--------------------------------------------|--------------------------------------|
|Student Organization Executive|No login needed — uses email to identify    |Submit and resubmit registration forms|
|Officer                       |Firebase Auth (Google account @my.xu.edu.ph)|Access login.html to submit form      |
|SACDEV Admin                  |Firebase Auth (Google account @xu.edu.ph)   |Full admin panel access               |

-----

## 2. File Structure

```
deploy-testing/
├── backend/
│   ├── server.js            ← The entire backend. All 1,600 lines.
│   ├── serviceAccountKey.json ← Firebase credentials (never commit this)
│   └── package.json
├── frontend/
│   ├── index.html           ← Landing page with links to register/officer
│   ├── registration.html    ← Multi-step org registration form
│   ├── login.html           ← Officer/org login page (Firebase Auth)
│   ├── admin.html           ← Admin panel (protected)
│   ├── admin-log.html       ← Activity log viewer
│   ├── officer-request.html ← Officer authorization request form
│   ├── script.js            ← Registration form logic (2,678 lines)
│   ├── admin.js             ← Admin panel logic (2,036 lines)
│   ├── admin.css            ← Admin panel styles
│   ├── firebase-auth.js     ← Firebase client-side auth setup
│   └── style.css            ← Main site styles
├── docker-compose.yml       ← Defines the two containers
├── Dockerfile.backend       ← How to build the backend container
├── Dockerfile.frontend      ← How to build the frontend (Nginx) container
├── nginx.conf               ← Nginx config: serves files, proxies /api/
└── .env                     ← Secret environment variables (never commit)
```

-----

## 3. How the System Starts

### When Docker Compose runs:

**Step 1 — Backend starts (`server.js` line 1–165):**

```
Load .env file manually (no dotenv package)
  → reads GMAIL_APP_PASSWORD, ALLOWED_ORIGIN, FIREBASE_SERVICE_ACCOUNT
  → sets them as process.env variables

Initialize Express app
  → app.use(cors(...))    — allows cross-origin requests from allowed domains
  → app.use(express.json({ limit: '10mb' })) — parses JSON request bodies up to 10MB

Initialize Firebase Admin SDK
  → reads serviceAccountKey.json (or FIREBASE_SERVICE_ACCOUNT env var)
  → connects to Firestore database
  → sets FIRESTORE_PREFER_REST=1 (forces REST API, avoids gRPC issues in Docker)

Firestore health check
  → writes { ok: true } to _health/ping collection
  → if this fails, full error report printed to console (wrong project, bad credentials, etc.)

Initialize Nodemailer mailer
  → SMTP: smtp.gmail.com port 587 (TLS)
  → auth: mardompaurysacdev@gmail.com + GMAIL_APP_PASSWORD from .env

Set up rate limiter (in-memory Map)
  → cleans expired entries every 30 minutes

Register all 24 API routes

Start listening on port 5000
```

**Step 2 — Frontend starts:**

```
Nginx serves all .html, .css, .js files from /usr/share/nginx/html
Any request to /api/* is forwarded to backend:5000 (same Docker network)
```

-----

## 4. Middleware

Middleware is code that runs **before** the actual route handler. Think of it as a checkpoint.

### 4.1 CORS (Cross-Origin Resource Sharing)

```javascript
// server.js line ~27
app.use(cors({ origin: function(origin, callback) { ... } }))
```

- Checks if the request comes from an allowed origin
- Allowed: `process.env.ALLOWED_ORIGIN`, `localhost`, `localhost:5000`, `localhost:3000`
- Requests with no origin (Postman, curl, server-to-server) are also allowed
- **For defense:** CORS protects the API from being called by random websites

### 4.2 JSON Body Parser

```javascript
app.use(express.json({ limit: '10mb' }))
```

- Parses incoming request bodies as JSON
- 10MB limit — large enough for form data with many fields, small enough to prevent abuse

### 4.3 `requireAdmin` — Admin Authentication Gate

```javascript
// server.js line ~67
async function requireAdmin(req, res, next) {
```

Every admin API route uses this. Here is what it does step by step:

1. Reads the `Authorization: Bearer <token>` header from the request
1. If no token → returns `401 Unauthorized`
1. Calls `admin.auth().verifyIdToken(token)` — Firebase checks if the token is real and not expired
1. If the token’s email does **not** end in `@xu.edu.ph` → returns `403 Forbidden`
1. If everything passes → sets `req.adminEmail = decoded.email` and calls `next()` to continue

**For defense:** Admin authentication is entirely server-side. The frontend cannot bypass it because it never even controls the check — Firebase verifies the token directly on the server.

### 4.4 Rate Limiter

```javascript
// server.js line ~183
function rateLimit({ windowMs, max, message }) {
```

- Tracks requests per IP address per route in a `Map` in memory
- Applied to `POST /submit` (max 5 per hour) and `POST /officer-request` (max 3 per hour)
- When limit is hit → returns `429 Too Many Requests`
- Resets automatically after the time window expires
- **Limitation:** Resets on server restart (no Redis)

### 4.5 `logActivity` — Activity Logger

```javascript
// server.js line ~88
async function logActivity({ adminEmail, action, targetType, targetId, targetName, details }) {
```

- Called **after** admin actions complete (after response is sent)
- Writes a document to the `activityLog` Firestore collection
- Fields: adminEmail, action, targetType, targetId, targetName, details, timestamp
- Never blocks the response — runs after `res.json()`

### 4.6 `escapeHtml` — XSS Prevention

```javascript
// server.js line ~103
function escapeHtml(str) {
```

- Converts `<`, `>`, `&`, `"`, `'` into their HTML entity equivalents
- Applied to all admin-typed text before it’s embedded in email HTML templates
- Prevents an admin who types `<script>alert(1)</script>` in a rejection reason from injecting code into emails

### 4.7 `sanitize` — Form Data Cleaner

```javascript
// server.js line ~251
const sanitize = (obj) => { ... }
```

- Recursively walks the entire submission object
- **Strips:** any string value starting with `data:` (raw base64 blobs)
- **Strips:** any field whose key matches `img_|Photo|Signature|Logo|preview` if the value is a long string not starting with `https://` (prevents raw base64 images from being stored in Firestore)
- **Fixes:** keys starting with `__` (strips the prefix)
- **Normalizes:** nested arrays (2D arrays from table fields) into objects like `{ c0: val, c1: val, ... }` because Firestore cannot store arrays of arrays

-----

## 5. Firestore Database

Firestore is a NoSQL document database. Data is organized into **collections** → **documents**.

### Collection: `submissions`

The most important collection. One document per org registration.

|Field            |Type     |Set When                |Value                                                        |
|-----------------|---------|------------------------|-------------------------------------------------------------|
|`org`            |string   |On submit               |Organization name                                            |
|`orgName`        |string   |On submit               |Same as org (legacy duplicate)                               |
|`orgEmail`       |string   |On submit               |Organization email (lowercase)                               |
|`email`          |string   |On submit               |Officer email (lowercase)                                    |
|`cluster`        |string   |On submit               |e.g. “Business”, “Natural Sciences…”                         |
|`status`         |string   |On submit / admin update|`pending` / `approved` / `rejected` / `revision`             |
|`published`      |boolean  |On admin publish        |`true` / `false`                                             |
|`academicYear`   |string   |On submit (calculated)  |e.g. `"2025-2026"`                                           |
|`revisionNotes`  |string   |Admin requests revision |The revision instructions                                    |
|`rejectionReason`|string   |Admin rejects           |The rejection reason                                         |
|`officers`       |array    |On submit               |Array of officer objects `{ name, studentId, position, ... }`|
|`createdAt`      |timestamp|On submit               |Server timestamp                                             |
|`updatedAt`      |timestamp|On status change        |Server timestamp                                             |
|`resubmittedAt`  |timestamp|On resubmit             |Server timestamp                                             |
|`publishedAt`    |timestamp|On publish              |Server timestamp                                             |
|All form fields  |mixed    |On submit               |Everything the org filled in the multi-step form             |

**Subcollection: `submissions/{id}/notes`**

|Field      |Type     |Description             |
|-----------|---------|------------------------|
|`text`     |string   |The note content        |
|`author`   |string   |Admin email who wrote it|
|`createdAt`|timestamp|When written            |

### Collection: `officerRequests`

One document per officer authorization request.

|Field            |Type     |Description                           |
|-----------------|---------|--------------------------------------|
|`fullName`       |string   |Officer’s full name                   |
|`email`          |string   |@my.xu.edu.ph email (lowercase)       |
|`contact`        |string   |Phone number                          |
|`organization`   |string   |Organization name                     |
|`position`       |string   |President / Vice President / Secretary|
|`status`         |string   |`pending` / `approved` / `rejected`   |
|`reviewedAt`     |timestamp|When admin acted on it                |
|`reviewedBy`     |string   |Admin email who reviewed              |
|`rejectionReason`|string   |If rejected                           |
|`createdAt`      |timestamp|When submitted                        |

### Collection: `conflicts`

One document per detected officer conflict.

|Field          |Type     |Description                        |
|---------------|---------|-----------------------------------|
|`studentId`    |string   |The conflicting student’s ID       |
|`studentName`  |string   |Student’s name                     |
|`submissionId1`|string   |ID of first conflicting submission |
|`orgName1`     |string   |First org name                     |
|`orgEmail1`    |string   |First org email                    |
|`position1`    |string   |Position in org 1                  |
|`submissionId2`|string   |ID of second conflicting submission|
|`orgName2`     |string   |Second org name                    |
|`orgEmail2`    |string   |Second org email                   |
|`position2`    |string   |Position in org 2                  |
|`notified`     |boolean  |Whether email was sent             |
|`notifiedAt`   |timestamp|When notified                      |
|`resolved`     |boolean  |Whether marked resolved            |
|`resolvedAt`   |timestamp|When resolved                      |
|`createdAt`    |timestamp|When detected                      |

### Collection: `drafts`

One document per org (keyed by email). Stores saved progress.

|Field    |Type     |Description                  |
|---------|---------|-----------------------------|
|`email`  |string   |The org’s email (document ID)|
|`data`   |object   |The entire partial form data |
|`savedAt`|timestamp|Last save time               |

### Collection: `activityLog`

One document per admin action.

|Field       |Type     |Description                                           |
|------------|---------|------------------------------------------------------|
|`adminEmail`|string   |Who did it                                            |
|`action`    |string   |e.g. `submission_approved`, `officer_request_rejected`|
|`targetType`|string   |`submission` or `officerRequest`                      |
|`targetId`  |string   |The Firestore document ID of the thing acted on       |
|`targetName`|string   |Human-readable name (org name, officer name)          |
|`details`   |string   |Optional extra info (reason, etc.)                    |
|`timestamp` |timestamp|When it happened                                      |

### Collection: `settings`

Single document: `submissionWindow`

|Field      |Type     |Description                                                                    |
|-----------|---------|-------------------------------------------------------------------------------|
|`isOpen`   |number   |`1` = open, `0` = closed (integer, not boolean — Firestore drops boolean false)|
|`openDate` |string   |ISO date string for when submissions open                                      |
|`closeDate`|string   |ISO date string for when submissions close                                     |
|`updatedAt`|timestamp|Last update time                                                               |
|`updatedBy`|string   |Admin email who last changed it                                                |

### Collection: `_health`

Single document: `ping`. Only used at startup to verify Firestore is writable.

-----

## 6. All API Routes Explained

### Format: `METHOD /route` — Auth — What It Does

-----

### `GET /ping`

**Auth:** None  
**What it does:** Returns `{ ok: true, timestamp }`. Used by cron services (e.g. cron-job.org) to ping the server every 10 minutes and prevent it from sleeping on free hosting tiers like Render.

-----

### `POST /submit`

**Auth:** None (rate limited: 5 per IP per hour)  
**What it does:**

1. Checks submission window — reads `settings/submissionWindow` from Firestore. If `isOpen === 0`, or if current date is before `openDate` or after `closeDate`, returns 403 with a message.
1. Sanitizes the entire request body using the `sanitize()` function.
1. Normalizes emails to lowercase.
1. Calculates academic year: if current month ≥ August, AY = this year to next year. Otherwise, previous year to this year.
1. Writes the sanitized data + `status: 'pending'` + `published: false` + `academicYear` to `submissions` collection.
1. Runs `detectAndStoreConflicts()` in the background (does not wait for it).
1. Sends a submission confirmation email to `orgEmail`.
1. Returns `{ message, id }`.

-----

### `GET /submissions`

**Auth:** `requireAdmin`  
**What it does:**

1. Fetches all documents from `submissions` collection.
1. For each document, finds “siblings” — other submissions with the same org name in the same academic year. If any exist, marks `isDuplicate: true`.
1. Sorts by `createdAt` descending (newest first).
1. Returns the full array.

**Why the duplicate check matters:** The admin table needs to know if a submission is a duplicate so the delete button is shown inside the detail modal.

-----

### `GET /submissions/:id/data`

**Auth:** None (public, but restricted)  
**What it does:**

1. Fetches the submission document by ID.
1. If status is **not** `revision` → returns 403. This means you can only get the raw form data for a submission that the admin has explicitly marked “for revision.”
1. Returns the full document.

**Why:** This is how the registration form prefills with existing data when an org needs to resubmit. Security: they can only see data for their own submission when it’s in revision status.

-----

### `PATCH /submissions/:id/status`

**Auth:** `requireAdmin`  
**What it does:**

1. Validates status is one of: `pending`, `approved`, `rejected`, `revision`.
1. Updates `status`, `updatedAt` in Firestore. Also sets `rejectionReason` or `revisionNotes` if reason provided.
1. Sends the appropriate email:
- `approved` → sends approval email
- `rejected` → sends rejection email with reason block
- `revision` → sends revision request email with notes block
1. Calls `logActivity()` — **but only if status is not `pending`** (we don’t log “set to pending” actions).
1. Returns `{ message, id }`.

-----

### `PATCH /submissions/:id/publish`

**Auth:** `requireAdmin`  
**What it does:**

1. Reads `published` from request body. Any value other than `false` is treated as `true`.
1. Updates `published` field + sets or deletes `publishedAt` timestamp.
1. Logs action as `submission_published` or `submission_unpublished`.
1. Returns `{ message, id }`.

-----

### `GET /org-plans/:orgName`

**Auth:** None (public)  
**What it does:**

1. Decodes the org name from the URL.
1. Builds multiple name variants (handles “XU – JMA” or “JMA (Junior Marketing Association)” formats).
1. For each variant, queries Firestore for a submission where `org === variant` AND `published === true`.
1. Extracts strategic plan tables: `table_bodyOrgDev`, `table_bodyStudServ`, `table_bodyCommInv`.
1. Normalizes each table row (handles both array format and object format from old/new form versions).
1. Returns `{ published, orgName, cluster, mission, vision, plans: { orgDev, studServ, commInv } }`.

-----

### `GET /submission-status`

**Auth:** None (public)  
**Query param:** `?email=orgname@email.com`  
**What it does:**

1. Tries exact Firestore query for `email === lowercaseEmail`.
1. If nothing found, tries `orgEmail === lowercaseEmail`.
1. If still nothing, does a full collection scan and compares case-insensitively (catches old docs with inconsistent case).
1. Sorts results by `createdAt` descending, returns the most recent.
1. Returns `{ found, id, status, org, revisionNotes, rejectionReason, ... }`.

**Why the fallback scan:** Early submissions may have been stored with mixed-case emails before email normalization was added.

-----

### `POST /officer-request`

**Auth:** None (rate limited: 3 per IP per hour)  
**What it does:**

1. Validates all fields are present.
1. Validates position is one of: `President`, `Vice President`, `Secretary`.
1. Validates email ends with `@my.xu.edu.ph`.
1. Checks if a `pending` or `approved` request already exists for this email in `officerRequests`. If yes, returns 409 Conflict.
1. Writes new document to `officerRequests` with `status: 'pending'`.
1. Sends confirmation email to officer’s email.
1. Returns `{ message }`.

-----

### `GET /check-officer-approval`

**Auth:** None (public)  
**Query param:** `?email=officer@my.xu.edu.ph`  
**What it does:** Fetches all officer requests for that email, sorts by date, returns the status of the most recent one. Used by `login.html` to check if a user should be allowed to proceed.

-----

### `GET /officer-requests`

**Auth:** `requireAdmin`  
**What it does:** Fetches all documents from `officerRequests`, sorts newest first, converts timestamps to readable strings. Returns array.

-----

### `PATCH /officer-requests/:id/status`

**Auth:** `requireAdmin`  
**What it does:**

1. Validates status is `approved` or `rejected`.
1. Updates `status`, `reviewedAt`, `reviewedBy` in Firestore.
1. If **approved:**
- Tries to find existing Firebase Auth user with `admin.auth().getUserByEmail(email)`.
- If found → generates a password reset link.
- If not found → creates a new Firebase Auth account with a random temp password, then generates a password reset link.
- Sends approval email with the password reset link (valid 24 hours).
1. If **rejected:** Sends rejection email with reason.
1. Logs action. Returns `{ message, id }`.

-----

### `GET /conflicts`

**Auth:** `requireAdmin`  
**What it does:** Fetches all conflict documents ordered by `createdAt` descending, converts timestamps. Returns array.

-----

### `POST /conflicts/:id/notify`

**Auth:** `requireAdmin`  
**What it does:**

1. Fetches the conflict document.
1. Builds an HTML email with a table showing: student ID, student name, org 1 + position, org 2 + position.
1. Sends to both `orgEmail1` and `orgEmail2`.
1. Updates the conflict document: `notified: true`, `notifiedAt: timestamp`.
1. Returns `{ message, recipients }`.

-----

### `POST /conflicts/:id/resolve`

**Auth:** `requireAdmin`  
**What it does:** Updates conflict document: `resolved: true`, `resolvedAt: timestamp`. Returns `{ message, id }`.

-----

### `POST /rescan-conflicts`

**Auth:** `requireAdmin`  
**What it does:**

1. Fetches all submissions.
1. For every pair of submissions (i, j), compares executive officers by `studentId`.
1. If same `studentId` appears as exec officer in both, checks if a conflict document already exists.
1. If not, creates a new conflict document.
1. Returns `{ message, newCount }`.

-----

### `POST /submissions/:id/notes`

**Auth:** `requireAdmin`  
**What it does:** Creates a new document in `submissions/{id}/notes` subcollection with `text`, `author` (admin email), `createdAt`.

-----

### `GET /submissions/:id/notes`

**Auth:** `requireAdmin`  
**What it does:** Fetches all notes from `submissions/{id}/notes` subcollection, ordered by `createdAt` ascending. Returns array.

-----

### `PATCH /submissions/:id/resubmit`

**Auth:** None (public)  
**What it does:**

1. Fetches the submission. If status is not `revision` → returns 400.
1. Sanitizes the incoming request body (same sanitize function as `/submit`).
1. Normalizes emails.
1. Updates the Firestore document with `...sanitizedData` + `status: 'pending'` + `revisionNotes: ''` + timestamps.
1. Preserves `academicYear`, `createdAt`, `published` from original document (these cannot be overwritten).
1. Runs `detectAndStoreConflicts()` on the new data.
1. Returns `{ message, id }`.

-----

### `DELETE /submissions/:id`

**Auth:** `requireAdmin`  
**What it does:**

1. Fetches the submission.
1. Safety check: queries all submissions with the same `academicYear` and same org name. If no siblings found → returns 403 (cannot delete the only submission for an org).
1. Deletes all notes in `submissions/{id}/notes` subcollection.
1. Deletes all conflict documents where `submissionId1 === id` or `submissionId2 === id`.
1. Deletes the submission document itself.
1. Logs action. Returns `{ message, id }`.

-----

### `POST /save-progress`

**Auth:** None (public)  
**What it does:** Upserts (set, not add) a document in `drafts` collection using the email as the document ID. Stores the entire partial form data object. If called again with the same email, it overwrites.

-----

### `GET /load-progress`

**Auth:** None (public)  
**Query param:** `?email=org@email.com`  
**What it does:** Reads the draft document for that email. Returns `{ found: true, data: {...} }` or `{ found: false }`.

-----

### `GET /submission-window`

**Auth:** None (public)  
**What it does:** Reads `settings/submissionWindow`. Returns `{ isOpen, openDate, closeDate, updatedAt, updatedBy }`. If document doesn’t exist, returns `{ isOpen: true }` (defaults to open).

-----

### `PATCH /submission-window`

**Auth:** `requireAdmin`  
**What it does:**

1. Reads `isOpen`, `openDate`, `closeDate` from body.
1. Converts `isOpen` to integer: `0` or `1`. This is intentional — Firestore silently drops boolean `false` in some SDK versions.
1. Sets `settings/submissionWindow` document (overwrites entire document).
1. Returns `{ message, isOpen, updatedAt }`.

-----

### `GET /activity-log`

**Auth:** `requireAdmin`  
**Query param:** `?limit=100` (default 100)  
**What it does:** Fetches all activity log documents, sorts by timestamp descending in-memory, slices to limit. Returns array. Filters out `submission_pending` entries on the client side (admin-log.html).

-----

### `detectAndStoreConflicts(submissionId, submissionData)` *(internal function)*

Called automatically after every `POST /submit` and `PATCH /submissions/:id/resubmit`.

1. Extracts `officers` array from the submission.
1. Filters to only exec positions: `president`, `vice president`, `secretary`, `treasurer`, `auditor`.
1. Filters to only officers who have a `studentId`.
1. Fetches ALL other submissions.
1. For each exec officer, checks if any other submission lists the same `studentId` as an exec officer.
1. If a match is found, checks if a conflict document for this pair already exists.
1. If not, creates a conflict document.

-----

## 7. Email Templates

All emails are HTML. Sent by `mardompaurysacdev@gmail.com` using Gmail SMTP with App Password.

All user-supplied text (org names, reasons, officer names) goes through `escapeHtml()` before being embedded.

|Trigger                         |Subject                                                    |Recipient          |Contains                                            |
|--------------------------------|-----------------------------------------------------------|-------------------|----------------------------------------------------|
|`POST /submit` succeeds         |`[SACDEV SOMS] Submission Received — OrgName`              |orgEmail           |Status: Under Review, submission timestamp          |
|Status changed to `approved`    |`[SACDEV SOMS] Re-Registration Approved — OrgName`         |orgEmail           |Green approval box                                  |
|Status changed to `rejected`    |`[SACDEV SOMS] Re-Registration Not Approved — OrgName`     |orgEmail           |Red rejection box + reason (if provided)            |
|Status changed to `revision`    |`[SACDEV SOMS] Revision Required — OrgName`                |orgEmail           |Amber revision box + notes (if provided)            |
|`POST /officer-request` succeeds|`[SACDEV SOMS] Officer Access Request Received — OrgName`  |officer email      |Under review confirmation                           |
|Officer request approved        |`[SACDEV SOMS] Officer Access Approved — Set Your Password`|officer email      |Green approval + **password setup link** (24hr)     |
|Officer request rejected        |`[SACDEV SOMS] Officer Access Not Approved — OrgName`      |officer email      |Red rejection + reason                              |
|Conflict notify button clicked  |`[SACDEV SOMS] Officer Conflict Notice — StudentName`      |**both** org emails|Table with student ID, name, both orgs and positions|

**Email sending is non-blocking.** If an email fails to send, it only logs an error — it does **not** fail the API response. The status update still succeeds.

-----

## 8. Frontend Pages

### 8.1 `index.html` — Landing Page

|Element                            |Function Called                        |API Hit                               |What Happens                                                                         |
|-----------------------------------|---------------------------------------|--------------------------------------|-------------------------------------------------------------------------------------|
|“Proceed to Registration” button   |`proceedIfOpen('login.html')`          |`GET /api/submission-window`          |Checks if window is open. If yes, navigates to login.html. If no, shows closed modal.|
|“Request Officer Access” button    |`proceedIfOpen('officer-request.html')`|`GET /api/submission-window`          |Same window check, then navigates to officer-request.html                            |
|Org name search box                |`fetchOrgPlans(orgName)`               |`GET /api/org-plans/:orgName`         |Loads the org’s published strategic plans and displays them                          |
|“Check Status” email input + button|`checkStatus()`                        |`GET /api/submission-status?email=...`|Shows submission status, revision notes, or rejection reason                         |
|Closed modal “OK” button           |Closes modal                           |None                                  |Just hides the modal                                                                 |

### 8.2 `login.html` — Organization Login

**On page load:**

- Submission window guard runs: `fetch('/api/submission-window')` → if closed, `sessionStorage.setItem('sacdev_window_msg', reason)` + `window.location.replace('index.html')`

|Element              |Function Called                     |API Hit                                    |What Happens                                        |
|---------------------|------------------------------------|-------------------------------------------|----------------------------------------------------|
|Google Sign-in button|Firebase `signInWithPopup(provider)`|Firebase Auth (Google OAuth)               |Opens Google account picker                         |
|After Google sign-in |`checkOfficerApproval(email)`       |`GET /api/check-officer-approval?email=...`|Checks if this email has an approved officer request|
|If approved →        |Navigates to `registration.html`    |None                                       |Passes email via sessionStorage                     |
|If not approved →    |Shows “not authorized” message      |None                                       |Tells user to submit officer request first          |

### 8.3 `registration.html` — Multi-Step Registration Form

The form has multiple steps/sections. `script.js` manages all navigation.

**On page load:**

- Loads saved progress: `GET /api/load-progress?email=...`
- Checks submission window: `GET /api/submission-window`

|Element               |Function Called           |API Hit                             |What Happens                                                         |
|----------------------|--------------------------|------------------------------------|---------------------------------------------------------------------|
|Any input change      |`autoSave()` (debounced)  |`POST /api/save-progress`           |Saves entire form data after 2 seconds of inactivity                 |
|“Next” step button    |`nextStep()`              |None                                |Validates current step, shows next                                   |
|“Previous” step button|`prevStep()`              |None                                |Goes back one step                                                   |
|File upload input     |`uploadToCloudinary(file)`|Cloudinary API (direct from browser)|Uploads file, gets back a URL, stores URL in form data               |
|“Submit” button       |`submitForm()`            |`POST /api/submit`                  |Sends entire form data as JSON. On success: shows confirmation screen|

**How file uploads work:**
The file goes directly from the browser to Cloudinary (not through our backend). Cloudinary returns a URL. That URL is stored in the form data object. When the form is submitted, only the URL is sent to our backend — not the file itself.

### 8.4 `admin.html` — Admin Panel

**On page load:**

- Requires Firebase Auth login with `@xu.edu.ph` Google account
- `firebase-auth.js` sets up auth state listener
- `loadAllData()` is called: fetches submissions, officer requests, conflicts

**Registered Organizations table:**

|Element                |Function Called                |API Hit                                |What Happens                                |
|-----------------------|-------------------------------|---------------------------------------|--------------------------------------------|
|Any table row click    |`openDetailModal(submissionId)`|`GET /api/submissions` (already loaded)|Opens the detail modal for that submission  |
|Search/filter inputs   |`renderSubmissions(filtered)`  |None                                   |Client-side filtering of already-loaded data|
|Cluster filter dropdown|`filterSubmissions()`          |None                                   |Client-side filter                          |
|Status filter dropdown |`filterSubmissions()`          |None                                   |Client-side filter                          |

**Submission detail modal:**

|Element                                                      |Function Called                       |API Hit                                                                        |What Happens                                         |
|-------------------------------------------------------------|--------------------------------------|-------------------------------------------------------------------------------|-----------------------------------------------------|
|“✓ Approve” button                                           |`updateStatus(id, 'approved')`        |`PATCH /api/submissions/:id/status`                                            |Changes status to approved, sends approval email     |
|“↩ Request Revision” button                                  |`openRevisionModal(id)`               |None                                                                           |Opens modal to type revision notes                   |
|Revision modal “Submit”                                      |`submitRevision(id, notes)`           |`PATCH /api/submissions/:id/status` (body: `{status:'revision', reason:notes}`)|Sets status to revision, sends revision email        |
|“✕ Reject” button                                            |`openRejectModal(id)`                 |None                                                                           |Opens modal to type rejection reason                 |
|Reject modal “Confirm”                                       |`submitRejection(id, reason)`         |`PATCH /api/submissions/:id/status` (body: `{status:'rejected', reason}`)      |Sets status to rejected, sends rejection email       |
|“Publish Plans” button                                       |`togglePublish(id, true)`             |`PATCH /api/submissions/:id/publish`                                           |Sets published:true, makes plans visible on main site|
|“Unpublish Plans” button                                     |`togglePublish(id, false)`            |`PATCH /api/submissions/:id/publish`                                           |Sets published:false                                 |
|“Add Note” button                                            |`saveNote(id, text)`                  |`POST /api/submissions/:id/notes`                                              |Saves internal staff note                            |
|“🗑 Delete This Submission” button (only shown for duplicates)|`confirmDeleteSubmission(id, orgName)`|`DELETE /api/submissions/:id`                                                  |Shows confirm dialog, then deletes if confirmed      |
|Modal close (✕) button                                       |`closeDetailModal()`                  |None                                                                           |Closes the modal                                     |

**Officer Authorization Requests tab:**

|Element                       |Function Called                     |API Hit                                                                       |What Happens                                                          |
|------------------------------|------------------------------------|------------------------------------------------------------------------------|----------------------------------------------------------------------|
|“✓ Approve” button            |`approveOfficerRequest(id, btn)`    |`PATCH /api/officer-requests/:id/status` (body: `{status:'approved'}`)        |Approves request, creates Firebase account, sends password setup email|
|“✕ Reject” button             |`openOfficerRejectModal(id, name)`  |None                                                                          |Opens reject modal                                                    |
|Officer reject modal “Confirm”|`submitOfficerRejection(id, reason)`|`PATCH /api/officer-requests/:id/status` (body: `{status:'rejected', reason}`)|Rejects, sends rejection email                                        |

**Pending requests are shown in the main table. Reviewed (approved/rejected) requests collapse into the “Reviewed Requests” sub-section.**

**Officer Conflicts tab:**

|Element         |Function Called      |API Hit                          |What Happens                                          |
|----------------|---------------------|---------------------------------|------------------------------------------------------|
|“Notify” button |`notifyConflict(id)` |`POST /api/conflicts/:id/notify` |Sends conflict alert email to both organizations      |
|“Resolve” button|`resolveConflict(id)`|`POST /api/conflicts/:id/resolve`|Marks conflict as resolved                            |
|“Rescan” button |`rescanConflicts()`  |`POST /api/rescan-conflicts`     |Re-runs full conflict detection across all submissions|

**Active conflicts shown in main list. Resolved conflicts collapse into sub-section (same pattern as officer requests).**

**Report Panel:**

|Element                    |Function Called      |API Hit|What Happens                                                                                                                                                                    |
|---------------------------|---------------------|-------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|Report panel toggle button |`toggleReportPanel()`|None   |Opens/closes the sliding report panel                                                                                                                                           |
|Inside panel (auto-renders)|`buildReport()`      |None   |Uses already-loaded `allSubmissions` data. Renders doughnut chart (status breakdown), bar chart (by cluster), org submission status list (submitted on top, not submitted below)|
|“Export PDF” button        |`exportReportPDF()`  |None   |Uses jsPDF to render the report panel to PDF client-side                                                                                                                        |
|“Export CSV” button        |`exportCSV()`        |None   |Generates CSV from loaded data                                                                                                                                                  |

**Submission Window toggle:**

|Element                   |Function Called       |API Hit                       |What Happens       |
|--------------------------|----------------------|------------------------------|-------------------|
|Toggle switch (open/close)|`toggleWindow(isOpen)`|`PATCH /api/submission-window`|Updates isOpen flag|
|Open date input           |`saveWindowDates()`   |`PATCH /api/submission-window`|Saves open date    |
|Close date input          |`saveWindowDates()`   |`PATCH /api/submission-window`|Saves close date   |

### 8.5 `admin-log.html` — Activity Log

|Element             |Function Called|API Hit                |What Happens                                              |
|--------------------|---------------|-----------------------|----------------------------------------------------------|
|Page load           |`loadLogs()`   |`GET /api/activity-log`|Fetches all logs, filters out `submission_pending` entries|
|Search input        |`filterLog()`  |None                   |Client-side filter on admin email, org name, action       |
|Action type dropdown|`filterLog()`  |None                   |Client-side filter                                        |
|“Export CSV” button |`exportCSV()`  |None                   |Downloads current visible logs as CSV                     |

### 8.6 `officer-request.html` — Officer Authorization Request

**On page load:**

- Submission window guard: `GET /api/submission-window` → redirects to index if closed

|Element      |Function Called  |API Hit                    |What Happens                                                    |
|-------------|-----------------|---------------------------|----------------------------------------------------------------|
|Submit button|`submitRequest()`|`POST /api/officer-request`|Validates fields, sends request. Shows success or error message.|

-----

## 9. How Backend Connects to Everything

```
┌─────────────────────────────────────────────────────────┐
│                    BROWSER                              │
│   index.html / registration.html / login.html /         │
│   admin.html / officer-request.html / admin-log.html    │
│                                                         │
│   Files served by Nginx (port 80)                       │
│   API calls go to /api/* → Nginx proxies to backend     │
└───────────────────────┬─────────────────────────────────┘
                        │ HTTP JSON requests
                        ▼
┌─────────────────────────────────────────────────────────┐
│                  NGINX (port 80)                         │
│   GET /          → serves index.html                    │
│   GET /admin.html → serves admin.html                   │
│   GET /api/*     → proxy_pass to backend:5000           │
└───────────────────────┬─────────────────────────────────┘
                        │ proxied as HTTP to port 5000
                        ▼
┌─────────────────────────────────────────────────────────┐
│              EXPRESS.JS BACKEND (port 5000)              │
│                                                         │
│   Middleware:                                           │
│   CORS → JSON Parser → requireAdmin (admin routes only) │
│                                                         │
│   Routes:                                              │
│   POST /submit            → writes to Firestore        │
│   GET  /submissions       → reads from Firestore       │
│   PATCH /submissions/:id/status → Firestore + Gmail    │
│   POST /officer-request   → Firestore + Gmail          │
│   PATCH /officer-requests/:id/status → Firestore       │
│                                         + Firebase Auth │
│                                         + Gmail         │
│   POST /conflicts/:id/notify → Gmail                   │
│   GET  /activity-log      → reads from Firestore       │
│   PATCH /submission-window → Firestore                 │
└─────┬────────────────┬────────────────┬────────────────┘
      │                │                │
      ▼                ▼                ▼
┌──────────┐  ┌──────────────┐  ┌─────────────┐
│ Firebase  │  │  Cloudinary  │  │ Gmail SMTP  │
│ Firestore │  │  CDN         │  │ (Nodemailer)│
│           │  │              │  │             │
│ Reads and │  │ File uploads │  │ Sends HTML  │
│ writes    │  │ done directly│  │ emails on   │
│ all data  │  │ from browser │  │ status      │
│           │  │ (not backend)│  │ changes     │
└──────────┘  └──────────────┘  └─────────────┘
      │
      ▼
┌──────────────────┐
│ Firebase Auth    │
│                  │
│ verifyIdToken()  │
│ createUser()     │
│ generatePassword │
│ ResetLink()      │
└──────────────────┘
```

-----

## 10. CRUD Table

|Collection              |Create                                         |Read                                                  |Update                                                                                              |Delete                                |
|------------------------|-----------------------------------------------|------------------------------------------------------|----------------------------------------------------------------------------------------------------|--------------------------------------|
|`submissions`           |`POST /submit`                                 |`GET /submissions`, `GET /submissions/:id/data`       |`PATCH /submissions/:id/status`, `PATCH /submissions/:id/publish`, `PATCH /submissions/:id/resubmit`|`DELETE /submissions/:id`             |
|`submissions/{id}/notes`|`POST /submissions/:id/notes`                  |`GET /submissions/:id/notes`                          |—                                                                                                   |Deleted when parent submission deleted|
|`officerRequests`       |`POST /officer-request`                        |`GET /officer-requests`, `GET /check-officer-approval`|`PATCH /officer-requests/:id/status`                                                                |—                                     |
|`conflicts`             |`detectAndStoreConflicts()` (internal)         |`GET /conflicts`                                      |`POST /conflicts/:id/notify`, `POST /conflicts/:id/resolve`                                         |When parent submission deleted        |
|`drafts`                |`POST /save-progress` (upsert)                 |`GET /load-progress`                                  |`POST /save-progress` (overwrites)                                                                  |—                                     |
|`activityLog`           |`logActivity()` (internal, after admin actions)|`GET /activity-log`                                   |—                                                                                                   |—                                     |
|`settings`              |`PATCH /submission-window` (upsert)            |`GET /submission-window`                              |`PATCH /submission-window`                                                                          |—                                     |
|`_health`               |Server startup                                 |—                                                     |—                                                                                                   |—                                     |

-----

## 11. Security Model

### Authentication Layers

```
Layer 1 — Firebase ID Token
  Every admin API call must include: Authorization: Bearer <Firebase_ID_token>
  Backend calls admin.auth().verifyIdToken(token) — Firebase validates it server-side
  If token invalid or expired → 401

Layer 2 — Domain Restriction
  Token is valid but email doesn't end in @xu.edu.ph → 403
  This check is server-side — cannot be bypassed from the browser

Layer 3 — Submission Window Guard (Frontend)
  login.html and officer-request.html call GET /submission-window on load
  If closed → redirect to index.html before user sees anything
  This is defense-in-depth — the backend also checks the window on POST /submit

Layer 4 — Rate Limiting
  POST /submit: 5 requests per IP per hour
  POST /officer-request: 3 requests per IP per hour
  Prevents automated spam submissions
```

### What’s Public (No Auth Required)

- `GET /ping`
- `GET /submission-window`
- `GET /submission-status`
- `GET /submissions/:id/data` (restricted to revision-status only)
- `GET /org-plans/:orgName` (restricted to published only)
- `GET /check-officer-approval`
- `POST /submit` (rate limited)
- `POST /officer-request` (rate limited)
- `POST /save-progress`
- `GET /load-progress`
- `PATCH /submissions/:id/resubmit`

### What Requires Admin Auth

Everything else — all GET /submissions, all status updates, all delete, all conflict management, all officer request review, activity log, submission window management.

### Data Sanitization

- All base64 blobs stripped from submission data before Firestore write
- Nested arrays converted to objects (Firestore limitation)
- All user text passed through `escapeHtml()` before embedding in emails
- Emails normalized to lowercase before storage and comparison

-----

## 12. Design Decisions

|Decision                                           |Why                                                                                                                                                                                                          |
|---------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|**No dotenv package**                              |`fs.readFileSync` on the `.env` file achieves the same thing without an extra dependency. Simpler.                                                                                                           |
|**`isOpen` stored as `0`/`1` integer, not boolean**|Firestore’s Node SDK silently drops boolean `false` when using `update()`. Storing `0` is reliable.                                                                                                          |
|**In-memory rate limiter (no Redis)**              |Keeps the system self-contained with no extra infrastructure. Acceptable limitation for a single-server university deployment.                                                                               |
|**Cloudinary uploads direct from browser**         |If uploads went through our backend, we’d need to handle file streaming, multipart parsing, and memory. Direct-to-Cloudinary keeps the backend light and fast.                                               |
|**Firebase Auth for admin only, not for orgs**     |Org executives don’t have university Google accounts. The system identifies them by email in the form instead.                                                                                               |
|**activityLog never logs “set to pending”**        |Resubmission automatically resets status to pending. Logging every resubmit as “set to pending” would pollute the log with noise.                                                                            |
|**Conflict detection runs in background**          |`detectAndStoreConflicts().catch(e => ...)` — doesn’t block the submit response. Conflict detection takes time (full collection fetch); making the user wait for it would be bad UX.                         |
|**Case-insensitive email fallback scan**           |Early submissions before normalization had mixed-case emails. The fallback scan catches these without requiring a database migration.                                                                        |
|**Delete only allowed for duplicates**             |Safety guard — prevents admins from accidentally deleting the only submission for an org. Must have a sibling to allow delete.                                                                               |
|**Notes stored as Firestore subcollection**        |Keeps note history clean and queryable without bloating the main submission document.                                                                                                                        |
|**Docker Compose, two containers**                 |Frontend and backend are decoupled. Backend can be updated without rebuilding the Nginx container and vice versa. Both share an internal Docker network — backend port 5000 should NOT be exposed externally.|
|**Static HTML/JS frontend (no React/Vue)**         |Keeps the deployment simple — no build step needed. Works well for a focused, single-purpose application.                                                                                                    |

-----

*End of SOMS System Guide*