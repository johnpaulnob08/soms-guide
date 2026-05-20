# SOMS — Complete System Guide

## OSA-SACDEV Student Organization Management System

### Xavier University – Ateneo de Cagayan

> **Purpose of this guide:** Your complete defense reference. Explains every file, every API route, every button, every database collection, and how everything connects — written in plain language.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [File Structure](#2-file-structure)
3. [How the System Starts](#3-how-the-system-starts)
4. [Middleware — Code That Runs Before Every Request](#4-middleware)
5. [Firestore Database — Collections and What They Store](#5-firestore-database)
6. [All API Routes Explained](#6-all-api-routes-explained)
7. [Email Templates — When They Send and What They Say](#7-email-templates)
8. [Frontend Pages — Every Button and What It Does](#8-frontend-pages)
9. [How Backend Connects to Everything](#9-how-backend-connects-to-everything)
10. [CRUD Table](#10-crud-table)
11. [Security Model](#11-security-model)
12. [Design Decisions](#12-design-decisions)

---

## 1. System Overview

SOMS can be run in two ways:

**Option A — Docker (local/self-hosted):**
```
Browser (User)
    ↓  HTTP request to port 80
[Frontend Container]  ←→  Nginx web server
    ↓  /api/* requests forwarded internally
[Backend Container]   ←→  Node.js + Express.js on port 5000
    ↓  reads/writes data
[Firebase Firestore]  ←→  Cloud NoSQL database (Google)
    ↓  sends emails
[Gmail API]           ←→  Email delivery via Gmail OAuth2 + googleapis
```

**Option B — Render / Railway (cloud deployment, current):**
```
Browser (User)
    ↓  HTTPS
[Frontend Service]    ←→  Static files served directly (e.g. xu-ateneorg.onrender.com)
    ↓  fetch() calls to backend URL
[Backend Service]     ←→  Node.js + Express.js (e.g. xu-ateneorg-backend.onrender.com)
    ↓  reads/writes data
[Firebase Firestore]  ←→  Cloud NoSQL database (Google)
    ↓  sends emails
[Gmail API]           ←→  Email delivery via Gmail OAuth2 + googleapis
```

**Three types of users:**

| User | How They Log In | What They Can Do |
|------|----------------|-----------------|
| Student Organization Executive | No login — uses email to identify | Check submission status on index.html |
| Officer | Firebase Auth (Google account @my.xu.edu.ph) | Log in via login.html to access and submit the registration form |
| SACDEV Admin | Firebase Auth (Google account @xu.edu.ph) | Full admin panel access |

---

## 2. File Structure

```
deploy-testing/
├── backend/
│   ├── server.js              ← The entire backend. All ~1,660 lines.
│   ├── serviceAccountKey.json ← Firebase credentials (never commit this)
│   ├── package.json
│   └── package-lock.json
├── frontend/
│   ├── index.html             ← Landing page: status check, org plan search
│   ├── registration.html      ← Multi-step org registration form
│   ├── login.html             ← Officer/org login page (Firebase Auth)
│   ├── admin.html             ← Admin panel (protected, @xu.edu.ph only)
│   ├── admin-log.html         ← Activity log viewer
│   ├── officer-request.html   ← Officer authorization request form
│   ├── script.js              ← Registration form logic
│   ├── admin.js               ← Admin panel logic
│   ├── admin.css              ← Admin panel styles
│   ├── firebase-auth.js       ← Firebase client-side auth setup
│   ├── styles.css             ← Main site styles
│   └── [logo/image assets]    ← xu_logo.png, xu_cent_logo.png, etc.
├── docker-compose.yml         ← Defines the two containers (for local/Docker deployment)
├── Dockerfile.backend         ← How to build the backend container
├── Dockerfile.frontend        ← How to build the frontend (Nginx) container
├── nginx.conf                 ← Nginx config: serves files, proxies /api/ to backend
└── .env                       ← Secret environment variables (never commit)
```

---

## 3. How the System Starts

### Environment Variables Required

| Variable | Description |
|----------|-------------|
| `GMAIL_USER` | The Gmail address used to send emails (e.g. `mardompaurysacdev@gmail.com`) |
| `GMAIL_CLIENT_ID` | From Google Cloud Console OAuth2 credentials |
| `GMAIL_CLIENT_SECRET` | From Google Cloud Console OAuth2 credentials |
| `GMAIL_REFRESH_TOKEN` | OAuth2 refresh token for Gmail API access |
| `FIREBASE_SERVICE_ACCOUNT` | Full contents of `serviceAccountKey.json` as a JSON string (used in cloud deployments) |
| `ALLOWED_ORIGIN` | The frontend URL allowed by CORS (e.g. `https://xu-ateneorg.onrender.com`) |
| `PORT` | Port the backend listens on (default: `5000`) |

### When the server starts (`server.js`):

```
Step 1 — Load environment
  Tries to read .env file from current or parent directory
  Falls back to host-provided environment variables (Render, Railway, Docker)

Step 2 — Initialize Express
  app.use(cors(...))               — restricts requests to ALLOWED_ORIGIN + localhost
  app.use(express.json({limit:'10mb'})) — parses JSON bodies up to 10MB

Step 3 — Initialize Firebase Admin SDK
  Reads FIREBASE_SERVICE_ACCOUNT env var (cloud) or serviceAccountKey.json (local)
  Connects to Firestore
  Sets FIRESTORE_PREFER_REST=1 (forces REST API, avoids gRPC issues in some environments)

Step 4 — Firestore health check
  Writes { ok: true } to _health/ping collection
  If this fails → prints full error report (wrong project, bad credentials, API not enabled)

Step 5 — Initialize Gmail API client
  Uses googleapis OAuth2 client with GMAIL_CLIENT_ID, GMAIL_CLIENT_SECRET, GMAIL_REFRESH_TOKEN
  No Nodemailer SMTP — emails are sent directly through Gmail API

Step 6 — Set up in-memory rate limiter
  Uses a Map to track requests per IP
  Cleans expired entries every 30 minutes

Step 7 — Register all API routes
  Serves frontend static files: app.use(express.static('../frontend'))
  Registers all 24+ API routes

Step 8 — Listen on PORT (default 5000)
```

### Docker deployment (local):

```
Nginx (port 80) serves frontend files
Any request to /api/* is proxied internally to backend:5000
Both containers run on the same Docker bridge network (sacdev_network)
```

### Cloud deployment (Render/Railway):

```
Frontend service serves static HTML/CSS/JS files directly
Frontend JS fetch() calls point to the backend service URL directly (no Nginx proxy)
CORS on the backend allows only the frontend's URL via ALLOWED_ORIGIN
```

---

## 4. Middleware

Middleware is code that runs **before** the actual route handler — like a checkpoint.

### 4.1 CORS (Cross-Origin Resource Sharing)

```javascript
app.use(cors({ origin: function(origin, callback) { ... } }))
```

- Checks if the request comes from an allowed origin
- Allowed: `process.env.ALLOWED_ORIGIN`, `localhost`, `localhost:5000`, `localhost:3000`
- Requests with no origin (Postman, curl, server-to-server) are also allowed
- **For defense:** CORS protects the API from being called by random websites. If `ALLOWED_ORIGIN` is not set in environment variables, all browser requests will be blocked.

### 4.2 JSON Body Parser

```javascript
app.use(express.json({ limit: '10mb' }))
```

- Parses incoming request bodies as JSON
- 10MB limit — large enough for form data with many fields, small enough to prevent abuse

### 4.3 `requireAdmin` — Admin Authentication Gate

```javascript
async function requireAdmin(req, res, next) { ... }
```

Every admin API route uses this. Step by step:

1. Reads the `Authorization: Bearer <token>` header from the request
2. If no token → returns `401 Unauthorized`
3. Calls `admin.auth().verifyIdToken(token)` — Firebase checks if the token is real and not expired
4. If the token's email does **not** end in `@xu.edu.ph` → returns `403 Forbidden`
5. If everything passes → sets `req.adminEmail = decoded.email` and calls `next()`

**For defense:** Admin authentication is entirely server-side. The frontend cannot bypass it — Firebase verifies the token directly on the server.

### 4.4 Rate Limiter

```javascript
function rateLimit({ windowMs, max, message }) { ... }
```

- Tracks requests per IP address per route using an in-memory `Map`
- Applied to `POST /submit` (max 5 per hour) and `POST /officer-request` (max 3 per hour)
- When limit is hit → returns `429 Too Many Requests`
- Resets automatically after the time window expires
- **Limitation:** Resets on server restart (no Redis/persistent store)

### 4.5 `logActivity` — Activity Logger

```javascript
async function logActivity({ adminEmail, action, targetType, targetId, targetName, details }) { ... }
```

- Called **after** admin actions complete (after `res.json()` is sent)
- Writes a document to the `activityLog` Firestore collection
- Fields: `adminEmail`, `action`, `targetType`, `targetId`, `targetName`, `details`, `timestamp`
- Never blocks the response — runs asynchronously after the reply

### 4.6 `escapeHtml` — XSS Prevention

```javascript
function escapeHtml(str) { ... }
```

- Converts `<`, `>`, `&`, `"`, `'` into their HTML entity equivalents
- Applied to all user-typed or admin-typed text before embedding in email HTML
- Prevents `<script>alert(1)</script>` in rejection reasons from injecting code into emails

### 4.7 `sanitize` — Form Data Cleaner

```javascript
const sanitize = (obj) => { ... }
```

- Recursively walks the entire submission object
- **Strips:** any string value starting with `data:` (raw base64 blobs)
- **Strips:** any field matching `img_|Photo|Signature|Logo|preview` if the value is a long non-URL string (prevents raw base64 images from being stored in Firestore)
- **Fixes:** keys starting with `__` (strips the prefix)
- **Normalizes:** 2D arrays (from table form fields) into objects like `{ c0: val, c1: val, ... }` because Firestore cannot store arrays of arrays

### 4.8 `_buildRawMessage` — Email Builder

```javascript
function _buildRawMessage({ to, subject, html }) { ... }
```

- Builds a raw RFC 2822 MIME email message
- Subject is encoded as `=?UTF-8?B?<base64>?=` (RFC 2047) to correctly handle special characters in org names (dashes, parentheses, accented letters, etc.)
- HTML body is encoded as base64 with `Content-Transfer-Encoding: base64` to preserve all Unicode characters
- Output is base64url-encoded for the Gmail API

---

## 5. Firestore Database

Firestore is a NoSQL document database. Data is organized into **collections** → **documents**.

### Collection: `submissions`

The most important collection. One document per org registration.

| Field | Type | Set When | Value |
|-------|------|----------|-------|
| `org` | string | On submit | Organization name |
| `orgName` | string | On submit | Same as org (legacy duplicate) |
| `orgEmail` | string | On submit | Organization email (lowercase) |
| `email` | string | On submit | Officer email (lowercase) |
| `cluster` | string | On submit | e.g. "Business", "Natural Sciences…" |
| `status` | string | On submit / admin update | `pending` / `approved` / `rejected` / `revision` |
| `published` | boolean | On admin publish | `true` / `false` |
| `academicYear` | string | On submit (calculated) | e.g. `"2025-2026"` |
| `revisionNotes` | string | Admin requests revision | The revision instructions |
| `rejectionReason` | string | Admin rejects | The rejection reason |
| `officers` | array | On submit | Array of `{ name, studentId, position, ... }` |
| `createdAt` | timestamp | On submit | Server timestamp |
| `updatedAt` | timestamp | On status change | Server timestamp |
| `resubmittedAt` | timestamp | On resubmit | Server timestamp |
| `publishedAt` | timestamp | On publish | Server timestamp |
| All form fields | mixed | On submit | Everything the org filled in the multi-step form |

**Subcollection: `submissions/{id}/notes`**

| Field | Type | Description |
|-------|------|-------------|
| `text` | string | The note content |
| `author` | string | Admin email who wrote it |
| `createdAt` | timestamp | When written |

### Collection: `officerRequests`

One document per officer authorization request.

| Field | Type | Description |
|-------|------|-------------|
| `fullName` | string | Officer's full name |
| `email` | string | @my.xu.edu.ph email (lowercase) |
| `contact` | string | Phone number |
| `organization` | string | Organization name |
| `position` | string | President / Vice President / Secretary |
| `status` | string | `pending` / `approved` / `rejected` |
| `reviewedAt` | timestamp | When admin acted on it |
| `reviewedBy` | string | Admin email who reviewed |
| `rejectionReason` | string | If rejected |
| `createdAt` | timestamp | When submitted |

### Collection: `conflicts`

One document per detected officer conflict.

| Field | Type | Description |
|-------|------|-------------|
| `studentId` | string | The conflicting student's ID |
| `studentName` | string | Student's name |
| `submissionId1` | string | ID of first conflicting submission |
| `orgName1` | string | First org name |
| `orgEmail1` | string | First org email |
| `position1` | string | Position in org 1 |
| `submissionId2` | string | ID of second conflicting submission |
| `orgName2` | string | Second org name |
| `orgEmail2` | string | Second org email |
| `position2` | string | Position in org 2 |
| `notified` | boolean | Whether email was sent |
| `notifiedAt` | timestamp | When notified |
| `resolved` | boolean | Whether marked resolved |
| `resolvedAt` | timestamp | When resolved |
| `createdAt` | timestamp | When detected |

### Collection: `drafts`

One document per org (keyed by email). Stores saved form progress.

| Field | Type | Description |
|-------|------|-------------|
| `email` | string | The org's email (used as document ID) |
| `data` | object | The entire partial form data |
| `savedAt` | timestamp | Last save time |

### Collection: `activityLog`

One document per admin action.

| Field | Type | Description |
|-------|------|-------------|
| `adminEmail` | string | Who did it |
| `action` | string | e.g. `submission_approved`, `officer_request_rejected` |
| `targetType` | string | `submission` or `officerRequest` |
| `targetId` | string | The Firestore document ID of the thing acted on |
| `targetName` | string | Human-readable name (org name, officer name) |
| `details` | string | Optional extra info (reason, etc.) |
| `timestamp` | timestamp | When it happened |

### Collection: `settings`

Single document: `submissionWindow`

| Field | Type | Description |
|-------|------|-------------|
| `isOpen` | number | `1` = open, `0` = closed (integer, not boolean — Firestore drops boolean false) |
| `openDate` | string | ISO date string for when submissions open |
| `closeDate` | string | ISO date string for when submissions close |
| `updatedAt` | timestamp | Last update time |
| `updatedBy` | string | Admin email who last changed it |

### Collection: `_health`

Single document: `ping`. Only used at startup to verify Firestore is writable.

---

## 6. All API Routes Explained

### Format: `METHOD /route` — Auth — What It Does

---

### `GET /ping`

**Auth:** None  
**What it does:** Returns `{ ok: true, ts: <timestamp> }`. Used by external cron services (e.g. cron-job.org) to ping the server every few minutes and prevent it from sleeping on free-tier cloud hosting.

---

### `POST /submit`

**Auth:** None (rate limited: 5 per IP per hour)  
**What it does:**

1. Checks submission window — reads `settings/submissionWindow` from Firestore. If `isOpen === 0`, or if current date is before `openDate` or after `closeDate`, returns 403.
2. Sanitizes the entire request body using `sanitize()`.
3. Normalizes emails to lowercase.
4. Calculates academic year: if current month ≥ August, AY = this year to next year. Otherwise, previous year to this year.
5. Writes sanitized data + `status: 'pending'` + `published: false` + `academicYear` to `submissions` collection.
6. Runs `detectAndStoreConflicts()` in the background (does not block response).
7. Sends a submission confirmation email to `orgEmail`.
8. Returns `{ message, id }`.

---

### `GET /submissions`

**Auth:** `requireAdmin`  
**What it does:**

1. Fetches all documents from `submissions` collection.
2. Sorts by `createdAt` descending (newest first).
3. Returns the full array.

---

### `GET /submissions/:id/data`

**Auth:** None (public, but restricted)  
**What it does:**

1. Fetches the submission document by ID.
2. If status is **not** `revision` → returns 403.
3. Returns the full document.

**Why:** This is how the registration form prefills with existing data when an org resubmits. Security: they can only see data for a submission that is explicitly in revision status.

---

### `PATCH /submissions/:id/status`

**Auth:** `requireAdmin`  
**What it does:**

1. Validates status is one of: `pending`, `approved`, `rejected`, `revision`.
2. Updates `status`, `updatedAt` in Firestore. Also sets `rejectionReason` or `revisionNotes` if reason provided.
3. Sends the appropriate email based on new status.
4. Logs the action via `logActivity()` — **only if status is not `pending`**.
5. Returns `{ message, id }`.

---

### `PATCH /submissions/:id/publish`

**Auth:** `requireAdmin`  
**What it does:**

1. Reads `published` from request body. Any value other than `false` is treated as `true`.
2. Updates `published` field + sets or deletes `publishedAt` timestamp.
3. Logs action as `submission_published` or `submission_unpublished`.
4. Returns `{ message, id }`.

---

### `GET /org-plans/:orgName`

**Auth:** None (public)  
**What it does:**

1. Decodes the org name from the URL.
2. Builds multiple name variants (handles formats like "XU – JMA" or "JMA (Junior Marketing Association)").
3. For each variant, queries Firestore for a submission where `org === variant` AND `published === true`.
4. Extracts strategic plan tables: `table_bodyOrgDev`, `table_bodyStudServ`, `table_bodyCommInv`.
5. Normalizes each table row (handles both array and object formats from different form versions).
6. Returns `{ published, orgName, cluster, mission, vision, plans: { orgDev, studServ, commInv } }`.

---

### `GET /submission-status`

**Auth:** None (public)  
**Query param:** `?email=org@email.com`  
**What it does:**

1. Tries exact Firestore query for `email === lowercaseEmail`.
2. If nothing found, tries `orgEmail === lowercaseEmail`.
3. If still nothing, does a full collection scan and compares case-insensitively (catches old docs with inconsistent case).
4. If multiple results, sorts by `createdAt` descending and returns the most recent.
5. Returns `{ found, id, status, org, revisionNotes, rejectionReason, ... }`.

**Why the fallback scan:** Early submissions may have mixed-case emails stored before normalization was added.

---

### `POST /officer-request`

**Auth:** None (rate limited: 3 per IP per hour)  
**What it does:**

1. Validates all fields are present.
2. Validates position is one of: `President`, `Vice President`, `Secretary`.
3. Validates email ends with `@my.xu.edu.ph`.
4. Checks if a `pending` or `approved` request already exists for this email. If yes → returns 409 Conflict.
5. Writes new document to `officerRequests` with `status: 'pending'`.
6. Sends confirmation email to the officer's email.
7. Returns `{ message }`.

---

### `GET /check-officer-approval`

**Auth:** None (public)  
**Query param:** `?email=officer@my.xu.edu.ph`  
**What it does:** Fetches all officer requests for that email, sorts by date, returns the status of the most recent one. Used by `login.html` to check if a user is allowed to proceed to registration.

---

### `GET /officer-requests`

**Auth:** `requireAdmin`  
**What it does:** Fetches all documents from `officerRequests`, sorts newest first, converts timestamps to readable strings. Returns array.

---

### `PATCH /officer-requests/:id/status`

**Auth:** `requireAdmin`  
**What it does:**

1. Validates status is `approved` or `rejected`.
2. Updates `status`, `reviewedAt`, `reviewedBy` in Firestore.
3. If **approved:**
   - Tries to find existing Firebase Auth user via `admin.auth().getUserByEmail(email)`.
   - If found → generates a password reset link.
   - If not found → creates new Firebase Auth account with a random temp password, then generates a password reset link.
   - Sends approval email with the password reset link (valid 24 hours).
4. If **rejected:** Sends rejection email with reason.
5. Logs action. Returns `{ message, id }`.

---

### `GET /conflicts`

**Auth:** `requireAdmin`  
**What it does:** Fetches all conflict documents ordered by `createdAt` descending, converts timestamps. Returns array.

---

### `POST /conflicts/:id/notify`

**Auth:** `requireAdmin`  
**What it does:**

1. Fetches the conflict document.
2. Builds an HTML email table showing: student ID, student name, org 1 + position, org 2 + position.
3. Sends to both `orgEmail1` and `orgEmail2`.
4. Updates conflict document: `notified: true`, `notifiedAt: timestamp`.
5. Returns `{ message, recipients }`.

---

### `POST /conflicts/:id/resolve`

**Auth:** `requireAdmin`  
**What it does:** Updates conflict document: `resolved: true`, `resolvedAt: timestamp`. Returns `{ message, id }`.

---

### `POST /rescan-conflicts`

**Auth:** `requireAdmin`  
**What it does:**

1. Fetches all submissions.
2. For every pair of submissions (i, j), compares executive officers by `studentId`.
3. If the same `studentId` appears as exec officer in both, checks if a conflict document already exists.
4. If not, creates a new conflict document.
5. Returns `{ message, newCount }`.

---

### `POST /submissions/:id/notes`

**Auth:** `requireAdmin`  
**What it does:** Creates a new document in `submissions/{id}/notes` subcollection with `text`, `author` (admin email), `createdAt`.

---

### `GET /submissions/:id/notes`

**Auth:** `requireAdmin`  
**What it does:** Fetches all notes from `submissions/{id}/notes` subcollection, ordered by `createdAt` ascending. Returns array.

---

### `PATCH /submissions/:id/resubmit`

**Auth:** None (public)  
**What it does:**

1. Fetches the submission. If status is not `revision` → returns 400.
2. Sanitizes the incoming request body (same `sanitize()` function as `/submit`).
3. Normalizes emails.
4. Updates the Firestore document with new data + `status: 'pending'` + `revisionNotes: ''` + timestamps.
5. Preserves `academicYear`, `createdAt`, `published` from the original document.
6. Runs `detectAndStoreConflicts()` on the new data.
7. Returns `{ message, id }`.

---

### `DELETE /submissions/:id`

**Auth:** `requireAdmin`  
**What it does:**

1. Fetches the submission.
2. Safety check: queries all submissions with the same `academicYear` and org name. If no siblings found → returns 403 (cannot delete the only submission for an org).
3. Deletes all notes in `submissions/{id}/notes` subcollection.
4. Deletes all conflict documents where `submissionId1 === id` or `submissionId2 === id`.
5. Deletes the submission document itself.
6. Logs action. Returns `{ message, id }`.

---

### `POST /save-progress`

**Auth:** None (public)  
**What it does:** Upserts a document in the `drafts` collection using the email as the document ID. Stores the entire partial form data. Calling again with the same email overwrites the previous draft.

---

### `GET /load-progress`

**Auth:** None (public)  
**Query param:** `?email=org@email.com`  
**What it does:** Reads the draft document for that email. Returns `{ found: true, data: {...} }` or `{ found: false }`.

---

### `GET /submission-window`

**Auth:** None (public)  
**What it does:** Reads `settings/submissionWindow`. Returns `{ isOpen, openDate, closeDate, updatedAt, updatedBy }`. If document doesn't exist, defaults to `{ isOpen: true }`.

---

### `PATCH /submission-window`

**Auth:** `requireAdmin`  
**What it does:**

1. Reads `isOpen`, `openDate`, `closeDate` from body.
2. Converts `isOpen` to integer: `0` or `1`. Intentional — Firestore silently drops boolean `false` in some SDK versions.
3. Sets `settings/submissionWindow` document (overwrites entire document).
4. Returns `{ message, isOpen, updatedAt }`.

---

### `GET /activity-log`

**Auth:** `requireAdmin`  
**Query param:** `?limit=100` (default 100)  
**What it does:** Fetches all activity log documents, sorts by timestamp descending in-memory, slices to limit. Returns array.

---

### `detectAndStoreConflicts(submissionId, submissionData)` *(internal function)*

Called automatically after every `POST /submit` and `PATCH /submissions/:id/resubmit`.

1. Extracts `officers` array from the submission.
2. Filters to only exec positions: `president`, `vice president`, `secretary`, `treasurer`, `auditor`.
3. Filters to only officers who have a `studentId`.
4. Fetches ALL other submissions.
5. For each exec officer, checks if any other submission lists the same `studentId` as an exec officer in those positions.
6. If a match is found and no conflict document exists yet, creates a new conflict document.

---

## 7. Email Templates

All emails are HTML. Sent by `mardompaurysacdev@gmail.com` using the **Gmail API** (OAuth2 via `googleapis` package — not SMTP/Nodemailer).

All user-supplied text (org names, reasons, officer names) goes through `escapeHtml()` before being embedded. Subjects and HTML bodies are UTF-8 base64 encoded to handle special characters in org names (dashes, parentheses, accented characters, etc.).

| Trigger | Subject | Recipient | Contains |
|---------|---------|-----------|----------|
| `POST /submit` succeeds | `[SACDEV SOMS] Submission Received - OrgName` | orgEmail | Status: Under Review, submission timestamp |
| Status changed to `approved` | `[SACDEV SOMS] Re-Registration Approved - OrgName` | orgEmail | Green approval box |
| Status changed to `rejected` | `[SACDEV SOMS] Re-Registration Not Approved - OrgName` | orgEmail | Red rejection box + reason (if provided) |
| Status changed to `revision` | `[SACDEV SOMS] Revision Required - OrgName` | orgEmail | Amber revision box + notes (if provided) |
| Status changed to `pending` (from approved/rejected) | `[SACDEV SOMS] Submission Under Review - OrgName` | orgEmail | Notification that submission is back under review |
| `POST /officer-request` succeeds | `[SACDEV SOMS] Officer Access Request Received - OrgName` | officer email | Under review confirmation |
| Officer request approved | `[SACDEV SOMS] Officer Access Approved - Set Your Password` | officer email | Green approval + **password setup link** (24hr) |
| Officer request rejected | `[SACDEV SOMS] Officer Access Not Approved - OrgName` | officer email | Red rejection + reason |
| Conflict notify button clicked | `[SACDEV SOMS] Officer Conflict Notice - StudentName` | **both** org emails | Table with student ID, name, both orgs and positions |

**Email sending is non-blocking.** If an email fails to send, it only logs an error — the API response still succeeds.

---

## 8. Frontend Pages

### 8.1 `index.html` — Landing Page

| Element | Function Called | API Hit | What Happens |
|---------|----------------|---------|--------------|
| "Proceed to Registration" button | `proceedIfOpen('login.html')` | `GET /submission-window` | Checks if window is open. If yes, navigates to login.html. If no, shows closed modal. |
| "Request Officer Access" button | `proceedIfOpen('officer-request.html')` | `GET /submission-window` | Same window check, then navigates to officer-request.html |
| Org name search box | `fetchOrgPlans(orgName)` | `GET /org-plans/:orgName` | Loads the org's published strategic plans and displays them |
| "Check Status" email input + button | `checkStatus()` | `GET /submission-status?email=...` | Shows submission status, revision notes, or rejection reason |
| Closed modal "OK" button | Closes modal | None | Just hides the modal |

### 8.2 `login.html` — Organization Login

**On page load:** Submission window guard runs → if closed, redirects to `index.html`.

| Element | Function Called | API Hit | What Happens |
|---------|----------------|---------|--------------|
| Google Sign-in button | Firebase `signInWithPopup(provider)` | Firebase Auth (Google OAuth) | Opens Google account picker |
| After Google sign-in | `checkOfficerApproval(email)` | `GET /check-officer-approval?email=...` | Checks if this email has an approved officer request |
| If approved → | Navigates to `registration.html` | None | Passes email via sessionStorage |
| If not approved → | Shows "not authorized" message | None | Tells user to submit officer request first |

### 8.3 `registration.html` — Multi-Step Registration Form

**On page load:** Loads saved progress (`GET /load-progress`) and checks submission window (`GET /submission-window`).

| Element | Function Called | API Hit | What Happens |
|---------|----------------|---------|--------------|
| Any input change | `autoSave()` (debounced) | `POST /save-progress` | Saves entire form data after 2 seconds of inactivity |
| "Next" step button | `nextStep()` | None | Validates current step, shows next |
| "Previous" step button | `prevStep()` | None | Goes back one step |
| File upload input | `uploadToCloudinary(file)` | Cloudinary API (direct from browser) | Uploads file, gets back a URL, stores URL in form data |
| "Submit" button | `submitForm()` | `POST /submit` | Sends entire form data as JSON. On success: shows confirmation screen |

**How file uploads work:** The file goes directly from the browser to Cloudinary (not through our backend). Cloudinary returns a URL. That URL is stored in the form data. When the form is submitted, only the URL is sent to our backend — not the file itself.

### 8.4 `admin.html` — Admin Panel

**On page load:** Requires Firebase Auth login with `@xu.edu.ph` Google account. `loadAllData()` fetches submissions, officer requests, and conflicts.

**Registered Organizations table:**

| Element | Function Called | API Hit | What Happens |
|---------|----------------|---------|--------------|
| Any table row click | `openDetailModal(submissionId)` | `GET /submissions` (already loaded) | Opens the detail modal for that submission |
| Search/filter inputs | `renderSubmissions(filtered)` | None | Client-side filtering of already-loaded data |
| Cluster filter dropdown | `filterSubmissions()` | None | Client-side filter |
| Status filter dropdown | `filterSubmissions()` | None | Client-side filter |

**Submission detail modal:**

| Element | Function Called | API Hit | What Happens |
|---------|----------------|---------|--------------|
| "✓ Approve" button | `updateStatus(id, 'approved')` | `PATCH /submissions/:id/status` | Changes status to approved, sends approval email |
| "↩ Request Revision" button | `openRevisionModal(id)` | None | Opens modal to type revision notes |
| Revision modal "Submit" | `submitRevision(id, notes)` | `PATCH /submissions/:id/status` | Sets status to revision, sends revision email |
| "✕ Reject" button | `openRejectModal(id)` | None | Opens modal to type rejection reason |
| Reject modal "Confirm" | `submitRejection(id, reason)` | `PATCH /submissions/:id/status` | Sets status to rejected, sends rejection email |
| "Publish Plans" button | `togglePublish(id, true)` | `PATCH /submissions/:id/publish` | Sets published:true, makes plans visible on main site |
| "Unpublish Plans" button | `togglePublish(id, false)` | `PATCH /submissions/:id/publish` | Sets published:false |
| "Add Note" button | `saveNote(id, text)` | `POST /submissions/:id/notes` | Saves internal staff note |
| "🗑 Delete" button (duplicates only) | `confirmDeleteSubmission(id, orgName)` | `DELETE /submissions/:id` | Shows confirm dialog, then deletes if confirmed |
| Modal close (✕) button | `closeDetailModal()` | None | Closes the modal |

**Officer Authorization Requests tab:**

| Element | Function Called | API Hit | What Happens |
|---------|----------------|---------|--------------|
| "✓ Approve" button | `approveOfficerRequest(id, btn)` | `PATCH /officer-requests/:id/status` | Approves request, creates Firebase account, sends password setup email |
| "✕ Reject" button | `openOfficerRejectModal(id, name)` | None | Opens reject modal |
| Officer reject modal "Confirm" | `submitOfficerRejection(id, reason)` | `PATCH /officer-requests/:id/status` | Rejects, sends rejection email |

**Officer Conflicts tab:**

| Element | Function Called | API Hit | What Happens |
|---------|----------------|---------|--------------|
| "Notify" button | `notifyConflict(id)` | `POST /conflicts/:id/notify` | Sends conflict alert email to both organizations |
| "Resolve" button | `resolveConflict(id)` | `POST /conflicts/:id/resolve` | Marks conflict as resolved |
| "Rescan" button | `rescanConflicts()` | `POST /rescan-conflicts` | Re-runs full conflict detection across all submissions |

**Report Panel:**

| Element | Function Called | API Hit | What Happens |
|---------|----------------|---------|--------------|
| Report panel toggle button | `toggleReportPanel()` | None | Opens/closes the sliding report panel |
| Inside panel (auto-renders) | `buildReport()` | None | Uses already-loaded data. Renders doughnut chart (status breakdown), bar chart (by cluster), org submission status list |
| "Export PDF" button | `exportReportPDF()` | None | Uses jsPDF to render the report panel to PDF client-side |
| "Export CSV" button | `exportCSV()` | None | Generates CSV from loaded data |

**Submission Window toggle:**

| Element | Function Called | API Hit | What Happens |
|---------|----------------|---------|--------------|
| Toggle switch (open/close) | `toggleWindow(isOpen)` | `PATCH /submission-window` | Updates isOpen flag |
| Open date input | `saveWindowDates()` | `PATCH /submission-window` | Saves open date |
| Close date input | `saveWindowDates()` | `PATCH /submission-window` | Saves close date |

### 8.5 `admin-log.html` — Activity Log

| Element | Function Called | API Hit | What Happens |
|---------|----------------|---------|--------------|
| Page load | `loadLogs()` | `GET /activity-log` | Fetches all logs, filters out `submission_pending` entries |
| Search input | `filterLog()` | None | Client-side filter on admin email, org name, action |
| Action type dropdown | `filterLog()` | None | Client-side filter |
| "Export CSV" button | `exportCSV()` | None | Downloads current visible logs as CSV |

### 8.6 `officer-request.html` — Officer Authorization Request

**On page load:** Submission window guard → redirects to index if closed.

| Element | Function Called | API Hit | What Happens |
|---------|----------------|---------|--------------|
| Submit button | `submitRequest()` | `POST /officer-request` | Validates fields, sends request. Shows success or error message. |

---

## 9. How Backend Connects to Everything

```
┌──────────────────────────────────────────────────────────────┐
│                        BROWSER                               │
│  index.html / registration.html / login.html /               │
│  admin.html / officer-request.html / admin-log.html          │
│                                                              │
│  [Docker]  Files served by Nginx (port 80)                   │
│            API calls go to /api/* → Nginx proxies to backend │
│                                                              │
│  [Cloud]   Files served by frontend service directly         │
│            API calls go directly to backend service URL      │
└──────────────────────┬───────────────────────────────────────┘
                       │ HTTP/HTTPS JSON requests
                       ▼
┌──────────────────────────────────────────────────────────────┐
│              EXPRESS.JS BACKEND (port 5000)                  │
│                                                              │
│  Middleware:                                                 │
│  CORS → JSON Parser → requireAdmin (admin routes only)       │
│                                                              │
│  Routes:                                                     │
│  POST /submit                → Firestore                     │
│  GET  /submissions           → Firestore                     │
│  PATCH /submissions/:id/status → Firestore + Gmail API       │
│  POST /officer-request       → Firestore + Gmail API         │
│  PATCH /officer-requests/:id/status → Firestore              │
│                                       + Firebase Auth        │
│                                       + Gmail API            │
│  POST /conflicts/:id/notify  → Gmail API                     │
│  GET  /activity-log          → Firestore                     │
│  PATCH /submission-window    → Firestore                     │
└──────┬──────────────────┬──────────────────┬────────────────┘
       │                  │                  │
       ▼                  ▼                  ▼
┌────────────┐   ┌──────────────┐   ┌──────────────────┐
│  Firebase  │   │  Cloudinary  │   │   Gmail API      │
│  Firestore │   │  CDN         │   │  (googleapis)    │
│            │   │              │   │  OAuth2 — not    │
│  Reads and │   │  File uploads│   │  SMTP/Nodemailer │
│  writes    │   │  done direct │   │                  │
│  all data  │   │  from browser│   │  Sends HTML      │
│            │   │  (not backend│   │  emails on all   │
│            │   │              │   │  status changes  │
└────────────┘   └──────────────┘   └──────────────────┘
       │
       ▼
┌──────────────────┐
│  Firebase Auth   │
│                  │
│  verifyIdToken() │
│  createUser()    │
│  generatePassword│
│  ResetLink()     │
└──────────────────┘
```

---

## 10. CRUD Table

| Collection | Create | Read | Update | Delete |
|-----------|--------|------|--------|--------|
| `submissions` | `POST /submit` | `GET /submissions`, `GET /submissions/:id/data` | `PATCH /submissions/:id/status`, `PATCH /submissions/:id/publish`, `PATCH /submissions/:id/resubmit` | `DELETE /submissions/:id` |
| `submissions/{id}/notes` | `POST /submissions/:id/notes` | `GET /submissions/:id/notes` | — | Deleted when parent submission deleted |
| `officerRequests` | `POST /officer-request` | `GET /officer-requests`, `GET /check-officer-approval` | `PATCH /officer-requests/:id/status` | — |
| `conflicts` | `detectAndStoreConflicts()` (internal) | `GET /conflicts` | `POST /conflicts/:id/notify`, `POST /conflicts/:id/resolve` | When parent submission deleted |
| `drafts` | `POST /save-progress` (upsert) | `GET /load-progress` | `POST /save-progress` (overwrites) | — |
| `activityLog` | `logActivity()` (internal, after admin actions) | `GET /activity-log` | — | — |
| `settings` | `PATCH /submission-window` (upsert) | `GET /submission-window` | `PATCH /submission-window` | — |
| `_health` | Server startup | — | — | — |

---

## 11. Security Model

### Authentication Layers

```
Layer 1 — Firebase ID Token
  Every admin API call must include: Authorization: Bearer <Firebase_ID_token>
  Backend calls admin.auth().verifyIdToken(token) — Firebase validates server-side
  If token invalid or expired → 401

Layer 2 — Domain Restriction
  Token is valid but email doesn't end in @xu.edu.ph → 403
  This check is server-side — cannot be bypassed from the browser

Layer 3 — Submission Window Guard (Frontend + Backend)
  login.html and officer-request.html call GET /submission-window on load
  If closed → redirect to index.html before user sees anything
  Backend also checks the window on POST /submit (defense in depth)

Layer 4 — Rate Limiting
  POST /submit: 5 requests per IP per hour
  POST /officer-request: 3 requests per IP per hour
  Prevents automated spam submissions
```

### What's Public (No Auth Required)

- `GET /ping`
- `GET /submission-window`
- `GET /submission-status`
- `GET /submissions/:id/data` (restricted to revision-status only)
- `GET /org-plans/:orgName` (restricted to published submissions only)
- `GET /check-officer-approval`
- `POST /submit` (rate limited)
- `POST /officer-request` (rate limited)
- `POST /save-progress`
- `GET /load-progress`
- `PATCH /submissions/:id/resubmit`

### What Requires Admin Auth

Everything else — `GET /submissions`, all status updates, delete, conflict management, officer request review, activity log, submission window management.

### Data Sanitization

- All base64 blobs stripped from submission data before Firestore write
- Nested arrays converted to objects (Firestore limitation)
- All user text passed through `escapeHtml()` before embedding in emails
- Emails normalized to lowercase before storage and comparison
- Email subjects and HTML bodies UTF-8 base64 encoded to handle special characters in org names

---

## 12. Design Decisions

| Decision | Why |
|----------|-----|
| **No dotenv package** | `fs.readFileSync` on the `.env` file achieves the same result without an extra dependency. Falls back to host environment variables seamlessly. |
| **Gmail API (OAuth2) instead of Nodemailer SMTP** | Gmail SMTP with App Passwords is being deprecated. OAuth2 via the Gmail API is the current recommended approach and more reliable for production use. |
| **`isOpen` stored as `0`/`1` integer, not boolean** | Firestore's Node SDK silently drops boolean `false` when using `update()`. Storing `0` is reliable. |
| **In-memory rate limiter (no Redis)** | Keeps the system self-contained with no extra infrastructure. Acceptable for a single-server university deployment. Resets on restart. |
| **Cloudinary uploads direct from browser** | If uploads went through our backend, we'd need multipart parsing and memory handling. Direct-to-Cloudinary keeps the backend light and fast. |
| **Firebase Auth for admin and officers only, not for org execs** | Org executives don't always have university Google accounts. The system identifies them by email entered in the form instead. |
| **`activityLog` never logs "set to pending"** | Resubmission automatically resets status to pending. Logging every resubmit as "set to pending" would pollute the log with noise. |
| **Conflict detection runs in background** | `detectAndStoreConflicts().catch(...)` doesn't block the submit response. Conflict detection requires a full collection fetch — making the user wait for it would be bad UX. |
| **Case-insensitive email fallback scan** | Early submissions before normalization had mixed-case emails. The fallback scan catches these without requiring a database migration. |
| **Delete only allowed for duplicates** | Safety guard — prevents admins from accidentally deleting the only submission for an org. A sibling must exist before delete is allowed. |
| **Notes stored as Firestore subcollection** | Keeps note history clean and queryable without bloating the main submission document. |
| **UTF-8 base64 encoding for email subjects and bodies** | Org names contain special characters (em dashes, parentheses, etc.). RFC 2047 encoded-word format (`=?UTF-8?B?...?=`) ensures subjects render correctly in all email clients. |
| **Deployment-agnostic backend** | The same `server.js` runs in Docker (with Nginx proxy), on Render, or on Railway without code changes — only environment variables differ. |
| **Static HTML/JS frontend (no React/Vue)** | Keeps deployment simple — no build step needed. Works well for a focused, single-purpose application. |

---

*End of SOMS System Guide*
