# SMIS V2 Backend — Postman Manual Test Plan

**Base URL:** `http://localhost:8000/api/v1`
**Content-Type:** `application/json` (unless noted as multipart)
**Auth:** Bearer JWT via `Authorization: Bearer <access_token>`

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Postman Automation Scripts](#postman-automation-scripts)
3. [P1. Auth](#p1-auth)
4. [P2. Users](#p2-users)
5. [P3. Geographic Hierarchy](#p3-geographic-hierarchy)
6. [P4. Academic Programs](#p4-academic-programs)
7. [P5. Announcements](#p5-announcements)
8. [P6. Students](#p6-students)
9. [P7. Enrollment — Public](#p7-enrollment--public)
10. [P8. Enrollment — Admin](#p8-enrollment--admin)
11. [P9. Academic Year Rollover](#p9-academic-year-rollover)
12. [P10. Realtime](#p10-realtime)
13. [P11. End-to-End Flows](#p11-end-to-end-flows)

---

## Getting Started

### 1. Start the backend

```bash
docker compose up -d
```

Verify the server is running:

```
GET http://localhost:8000/api/schema/swagger/
```

You should see the Swagger UI.

### 2. Import into Postman

1. Open Postman, click **Import**.
2. Paste the Swagger URL: `http://localhost:8000/api/schema/` (the raw OpenAPI JSON).
3. Postman auto-generates a collection from the schema.
4. Alternatively, create requests manually following the test cases below.

### 3. Create a Postman Environment

1. Click the **Environments** tab → **+** to create `SMIS Local`.
2. Add the variables listed below.
3. Select the environment in the top-right dropdown before running requests.

### 4. Seed test data

If the database is empty, populate it via Django Admin (`/admin/`) first:

1. Create a superuser: `docker compose exec web uv run python src/manage.py createsuperuser`
2. Login to `/admin/` and create: Central → Region → Division → District → School
3. Create AcademicProgram tree entries and link programs to the school via SchoolAcademicProgram
4. Create an AcademicTerm template and a SchoolAcademicTerm with open enrollment dates
5. Create SchoolEnrollmentSettings for at least one program

### 5. Run test cases

Each test case has a **TC-ID** (e.g. TC-1.1.1). Work through them in order — Auth first (to get tokens), then subsequent sections.

### 6. Documenting results in Google Docs

Create a **Google Spreadsheet** titled `SMIS V2 — API Test Results` with the following setup:

**Sheet 1: Test Log**

| Column | Description |
|---|---|
| **TC-ID** | Test case ID from this plan (e.g. TC-1.1.1) |
| **Section** | Section name (e.g. P1. Auth) |
| **Description** | Brief description of the test case |
| **Status** | `PASS` / `FAIL` / `BLOCKED` / `SKIPPED` |
| **Actual Result** | What actually happened (especially on failures) |
| **Response Code** | HTTP status code received |
| **Tested By** | Tester's name |
| **Date Tested** | Date of execution |
| **Environment** | e.g. `local`, `staging` |
| **Notes / Screenshots** | Links to screenshots, error messages, or observations |

**Sheet 2: Summary Dashboard**

Use formulas to build a live summary:

| Metric | Formula |
|---|---|
| Total Tests | `=COUNTA(A2:A)` |
| Passed | `=COUNTIF(D2:D,"PASS")` |
| Failed | `=COUNTIF(D2:D,"FAIL")` |
| Blocked | `=COUNTIF(D2:D,"BLOCKED")` |
| Skipped | `=COUNTIF(D2:D,"SKIPPED")` |
| Pass Rate | `=COUNTIF(D2:D,"PASS")/COUNTA(D2:D)*100` |

**Sheet 3: Bug Tracker** (for failures)

| Column | Description |
|---|---|
| **TC-ID** | Failing test case |
| **Severity** | `Critical` / `Major` / `Minor` |
| **Steps to Reproduce** | Copy from this plan + any extra context |
| **Expected vs Actual** | Side-by-side comparison |
| **Assigned To** | Developer responsible |
| **Fix Status** | `Open` / `In Progress` / `Fixed` / `Verified` |
| **Retest Date** | When retested after fix |

**Workflow:**

1. Before a test session, duplicate the sheet template so each session has its own timestamped copy.
2. Work through test cases in order. Fill each row as you execute.
3. For failures, take a screenshot of the Postman response and paste/link it in the Notes column.
4. After the session, update the Summary Dashboard and file bugs in Sheet 3.
5. Share the spreadsheet with the team — set **Commenter** access for devs, **Editor** for QA.

---

## Environment Variables (Postman)

Set these in a Postman environment so requests can reference them:

| Variable | Description | Example |
|---|---|---|
| `base_url` | API base URL | `http://localhost:8000/api/v1` |
| `access_token` | JWT access token (set after login) | `eyJ...` |
| `refresh_token` | JWT refresh token (set after login) | `eyJ...` |
| `school_id` | UUID of a test school | `e8187163-...` |
| `student_lrn` | A 12-digit LRN | `120345678901` |
| `submission_id` | UUID of an enrollment submission | `...` |
| `section_offering_id` | UUID of a section offering | `...` |
| `term_id` | UUID of a SchoolAcademicTerm | `...` |
| `program_id` | UUID of an AcademicProgram (leaf) | `...` |
| `user_id` | UUID of a target user | `...` |

---

## Postman Automation Scripts

Paste these scripts into the **Scripts** tab of each request to automate token management, assertions, and variable chaining. This eliminates manual copy-paste and enables Collection Runner execution.

### Collection-level Pre-request Script

Go to the **collection root** → Scripts → Pre-request. This runs before every request in the collection.

```javascript
// Auto-attach Bearer token to authenticated requests
const noAuthPaths = [
    '/auth/token',
    '/password/reset/request',
    '/password/reset/confirm',
    '/enrollment/submissions',  // POST is public
    '/schools/public/',
    '/students/disabilities',
    '/students/modalities',
    '/enrollment/settings',
    '/enrollment/section-offerings',
];

const url = pm.request.url.toString();
const isPublic = noAuthPaths.some(p => url.includes(p));
const isLogin = url.includes('/auth/token') && !url.includes('/refresh') && !url.includes('/verify') && !url.includes('/blacklist');

if (!isPublic || !isLogin) {
    const token = pm.environment.get('access_token');
    if (token) {
        pm.request.headers.add({
            key: 'Authorization',
            value: `Bearer ${token}`
        });
    }
}
```

### Login — Auto-save tokens (TC-1.1.1)

Paste in **POST /auth/token/** → Scripts → Post-response:

```javascript
if (pm.response.code === 200) {
    const body = pm.response.json();
    pm.environment.set('access_token', body.access);
    pm.environment.set('refresh_token', body.refresh);
    console.log('Tokens saved to environment.');

    // Assertions
    pm.test('Status is 200', () => pm.response.to.have.status(200));
    pm.test('Has access token', () => pm.expect(body.access).to.be.a('string'));
    pm.test('Has refresh token', () => pm.expect(body.refresh).to.be.a('string'));
}
```

### Login — Error assertions (TC-1.1.3 to TC-1.1.6)

Paste in the negative-case login requests:

```javascript
pm.test('Status is 401', () => pm.response.to.have.status(401));
pm.test('Error envelope present', () => {
    const body = pm.response.json();
    pm.expect(body).to.have.property('error');
});
```

### Token Refresh — Auto-save new tokens (TC-1.2.1)

Paste in **POST /auth/token/refresh/** → Post-response:

```javascript
if (pm.response.code === 200) {
    const body = pm.response.json();
    pm.environment.set('access_token', body.access);
    if (body.refresh) {
        pm.environment.set('refresh_token', body.refresh);
    }
    pm.test('Status is 200', () => pm.response.to.have.status(200));
    pm.test('New access token received', () => pm.expect(body.access).to.be.a('string'));
}
```

### GET /users/me/ — Profile assertions (TC-2.1.x)

```javascript
pm.test('Status is 200', () => pm.response.to.have.status(200));

const body = pm.response.json();

pm.test('Has user id', () => pm.expect(body.id).to.be.a('string'));
pm.test('Has profile_type', () => {
    pm.expect(['staff', 'student', 'none']).to.include(body.profile_type);
});
pm.test('Has scope object', () => {
    pm.expect(body.scope).to.have.property('is_global_admin');
    pm.expect(body.scope).to.have.property('group_names');
});

// Save user_id for later tests
pm.environment.set('user_id', body.id);
```

### Geographic Dropdowns — Generic list assertion

Reuse across all `GET .../centrals/`, `GET .../regions/`, etc.:

```javascript
pm.test('Status is 200', () => pm.response.to.have.status(200));

const body = pm.response.json();

pm.test('Has paginated structure', () => {
    pm.expect(body).to.have.property('count');
    pm.expect(body).to.have.property('results');
    pm.expect(body.results).to.be.an('array');
});

// Save first result ID for chaining (e.g. central_id → region filter)
if (body.results.length > 0) {
    const firstId = body.results[0].id;
    // Adjust the variable name per endpoint
    // pm.environment.set('central_id', firstId);
}
```

### Schools dropdown — Verify is_registered field (TC-3.5.1)

```javascript
pm.test('Status is 200', () => pm.response.to.have.status(200));

const body = pm.response.json();
if (body.results.length > 0) {
    pm.test('School has is_registered field', () => {
        pm.expect(body.results[0]).to.have.property('is_registered');
        pm.expect(body.results[0].is_registered).to.be.a('boolean');
    });

    // Save school_id for later tests
    pm.environment.set('school_id', body.results[0].id);
}
```

### Enrollment Submission — Save ID + assert (TC-7.3.1)

Paste in **POST /enrollment/submissions/** → Post-response:

```javascript
if (pm.response.code === 201) {
    const body = pm.response.json();

    pm.test('Status is 201', () => pm.response.to.have.status(201));
    pm.test('Status is pending', () => pm.expect(body.status).to.eql('pending'));
    pm.test('Has submission ID', () => pm.expect(body.id).to.be.a('string'));

    pm.environment.set('submission_id', body.id);
    console.log('Saved submission_id:', body.id);
}
```

### Enrollment Admission — Verify student created (TC-8.3.1)

Paste in **PATCH /enrollment/submissions/{id}/review/** → Post-response:

```javascript
if (pm.response.code === 200) {
    const body = pm.response.json();

    pm.test('Status is 200', () => pm.response.to.have.status(200));
    pm.test('Submission admitted', () => pm.expect(body.status).to.eql('admitted'));
    pm.test('Admitted student populated', () => {
        pm.expect(body.admitted_student).to.not.be.null;
    });
    pm.test('Reviewed at set', () => {
        pm.expect(body.reviewed_at).to.not.be.null;
    });
}
```

### Student Onboard — Save student IDs (TC-6.1.1, TC-6.2.1)

```javascript
if (pm.response.code === 201) {
    const body = pm.response.json();

    pm.test('Status is 201', () => pm.response.to.have.status(201));
    pm.test('Has created_count', () => pm.expect(body.created_count).to.be.above(0));
    pm.test('Has student_ids', () => pm.expect(body.student_ids).to.be.an('array'));

    if (body.student_ids.length > 0) {
        pm.environment.set('student_id', body.student_ids[0]);
    }
}
```

### Batch Photo Upload — Verify matches (TC-6.3.1)

```javascript
pm.test('Status is 200', () => pm.response.to.have.status(200));

const body = pm.response.json();

pm.test('Has matched count', () => pm.expect(body.matched).to.be.a('number'));
pm.test('Has not_found array', () => pm.expect(body.not_found).to.be.an('array'));
pm.test('Has errors array', () => pm.expect(body.errors).to.be.an('array'));
```

### Rollover — Dry run assertions (TC-9.1.1)

```javascript
pm.test('Status is 200', () => pm.response.to.have.status(200));

const body = pm.response.json();

pm.test('Has term_preview', () => pm.expect(body).to.have.property('term_preview'));
pm.test('Has sections plan', () => {
    pm.expect(body.sections).to.have.property('to_create');
    pm.expect(body.sections).to.have.property('carried_over');
    pm.expect(body.sections).to.have.property('needs_attention');
});
pm.test('No term_created (dry run)', () => {
    pm.expect(body).to.not.have.property('term_created');
});
```

### Generic Error Envelope assertion

Reuse on any request expecting a 400/401/404 error:

```javascript
pm.test('Error envelope format', () => {
    const body = pm.response.json();
    pm.expect(body).to.have.property('error');
    pm.expect(body.error).to.be.a('string');
    if (body.extra) {
        pm.expect(body.extra).to.be.an('object');
    }
});
```

### 401 Unauthorized assertion

Reuse on any request testing unauthenticated access:

```javascript
pm.test('Status is 401', () => pm.response.to.have.status(401));
pm.test('Error code is NOT_AUTHENTICATED', () => {
    const body = pm.response.json();
    pm.expect(body.extra.code).to.eql('NOT_AUTHENTICATED');
});
```

### 429 Rate Limit assertion

Reuse when testing throttle limits:

```javascript
pm.test('Status is 429', () => pm.response.to.have.status(429));
pm.test('Error code is THROTTLED', () => {
    const body = pm.response.json();
    pm.expect(body.extra.code).to.eql('THROTTLED');
});
```

---

### Using the Collection Runner

To execute all tests at once:

1. Click the **...** menu on your collection → **Run collection**.
2. Select the `SMIS Local` environment.
3. Order requests: Login first, then dependent requests.
4. Set **Delay** to `200ms` between requests (avoids rate limits on auth endpoints).
5. Click **Run SMIS V2 Backend**.
6. Review the pass/fail summary in the results tab.

**Tip:** Use `pm.execution.setNextRequest('Request Name')` in post-response scripts to chain requests in a specific order for E2E flows.

### Folder structure recommendation

Organize requests in Postman folders mirroring this document:

```
SMIS V2 Backend/
├── P1. Auth/
│   ├── Login (username)
│   ├── Login (email)
│   ├── Login (wrong password)
│   ├── Refresh Token
│   ├── Verify Token
│   └── Blacklist Token
├── P2. Users/
│   ├── GET /me (staff)
│   ├── GET /me (student)
│   ├── GET /me (unauthenticated)
│   ├── Password Reset Request
│   ├── Password Reset Confirm
│   └── Admin Password Reset
├── P3. Geographic Hierarchy/
│   ├── Centrals
│   ├── Regions
│   ├── Divisions
│   ├── Districts
│   └── Schools
├── ...
└── E2E Flows/
    ├── E1. Full Enrollment Cycle
    ├── E2. Student Onboarding
    ├── E3. Password Reset
    ├── E4. Academic Year Rollover
    └── E5. Login with Email
```

---

## P1. Auth

**Prefix:** `{{base_url}}/auth`

### P1.1 — `POST /auth/token/` — Obtain JWT Tokens

#### TC-1.1.1: Login with username + password

```
POST {{base_url}}/auth/token/
```

```json
{
  "username": "admin",
  "password": "your_password"
}
```

**Expected:** `200 OK`

```json
{
  "access": "<jwt_access_token>",
  "refresh": "<jwt_refresh_token>"
}
```

**Post-test script:** Save `access` → `{{access_token}}`, `refresh` → `{{refresh_token}}`.

---

#### TC-1.1.2: Login with email + password (new feature)

```
POST {{base_url}}/auth/token/
```

```json
{
  "username": "admin@example.com",
  "password": "your_password"
}
```

**Expected:** `200 OK` — same response as TC-1.1.1 (the `username` field accepts email).

---

#### TC-1.1.3: Wrong password

```json
{
  "username": "admin",
  "password": "wrong_password"
}
```

**Expected:** `401 Unauthorized`

```json
{
  "error": "No active account found with the given credentials",
  "extra": { "code": "AUTHENTICATION_FAILED" }
}
```

---

#### TC-1.1.4: Inactive user

Pre-condition: Set user `is_active=False` in admin.

```json
{
  "username": "inactive_user",
  "password": "correct_password"
}
```

**Expected:** `401 Unauthorized`

---

#### TC-1.1.5: Nonexistent user

```json
{
  "username": "doesnotexist",
  "password": "anything"
}
```

**Expected:** `401 Unauthorized`

---

#### TC-1.1.6: Empty body

```json
{}
```

**Expected:** `400 Bad Request` — validation error for required fields.

---

### P1.2 — `POST /auth/token/refresh/` — Refresh Access Token

#### TC-1.2.1: Valid refresh token

```
POST {{base_url}}/auth/token/refresh/
```

```json
{
  "refresh": "{{refresh_token}}"
}
```

**Expected:** `200 OK`

```json
{
  "access": "<new_access_token>",
  "refresh": "<new_refresh_token>"
}
```

Note: old refresh token is blacklisted after rotation (`ROTATE_REFRESH_TOKENS=True`).

---

#### TC-1.2.2: Expired refresh token

Use a refresh token older than 7 days.

**Expected:** `401 Unauthorized`

```json
{
  "error": "Token is invalid or expired",
  "extra": { "code": "AUTHENTICATION_FAILED" }
}
```

---

#### TC-1.2.3: Blacklisted refresh token

Use the refresh token from TC-1.2.1 again (already rotated).

**Expected:** `401 Unauthorized`

---

### P1.3 — `POST /auth/token/verify/` — Verify Token

#### TC-1.3.1: Valid access token

```
POST {{base_url}}/auth/token/verify/
```

```json
{
  "token": "{{access_token}}"
}
```

**Expected:** `200 OK` — empty body `{}`.

---

#### TC-1.3.2: Expired token

**Expected:** `401 Unauthorized`

---

#### TC-1.3.3: Garbage token

```json
{
  "token": "not.a.real.token"
}
```

**Expected:** `401 Unauthorized`

---

### P1.4 — `POST /auth/token/blacklist/` — Logout

#### TC-1.4.1: Blacklist refresh token

```
POST {{base_url}}/auth/token/blacklist/
```

```json
{
  "refresh": "{{refresh_token}}"
}
```

**Expected:** `200 OK`

---

#### TC-1.4.2: Refresh with blacklisted token

Use the same refresh token in `POST /auth/token/refresh/`.

**Expected:** `401 Unauthorized`

---

## P2. Users

**Prefix:** `{{base_url}}/users`

### P2.1 — `GET /users/me/` — Current User Profile & Scope

#### TC-2.1.1: Staff user

```
GET {{base_url}}/users/me/
Authorization: Bearer {{access_token}}
```

**Expected:** `200 OK`

```json
{
  "id": "uuid",
  "username": "admin",
  "email": "admin@example.com",
  "is_active": true,
  "is_staff": true,
  "is_superuser": true,
  "date_joined": "2026-01-01T00:00:00+00:00",
  "last_login": "...",
  "profile_image": null,
  "roles": ["admin"],
  "profile_type": "staff",
  "staff": {
    "id": "uuid",
    "employee_id": "EMP-001",
    "full_name": "John Doe",
    "first_name": "John",
    "last_name": "Doe",
    "middle_name": "",
    "suffix": "",
    "designation": "Principal",
    "assignment_level": "school",
    "school_id": "uuid",
    "school_name": "Test School"
  },
  "student": null,
  "scope": {
    "is_global_admin": true,
    "is_staff_scoped": true,
    "is_student": false,
    "central_id": "uuid",
    "region_id": "uuid",
    "division_id": "uuid",
    "district_id": "uuid",
    "school_id": "uuid",
    "group_names": ["admin"]
  }
}
```

**Verify:**
- `profile_type` is `"staff"`
- `staff` object is populated
- `student` is `null`
- `scope.group_names` contains the user's groups

---

#### TC-2.1.2: Student user

Login as a student user, then call `GET /users/me/`.

**Verify:**
- `profile_type` is `"student"`
- `student` object populated (id, lrn, full_name, school_id, school_name)
- `staff` is `null`
- `scope.is_student` is `true`

---

#### TC-2.1.3: Superuser with no profile

Create a user with `is_superuser=True` but no Staff or Student record.

**Verify:**
- `profile_type` is `"none"`
- `staff` and `student` are both `null`
- `scope.is_global_admin` is `true`

---

#### TC-2.1.4: Unauthenticated

```
GET {{base_url}}/users/me/
(no Authorization header)
```

**Expected:** `401 Unauthorized`

---

#### TC-2.1.5: Cache hit

Call `GET /users/me/` twice in rapid succession.

**Verify:** Both responses are identical. Second request should be faster (Redis cache, TTL 5 min).

---

### P2.2 — `POST /users/password/reset/request/` — Request Password Reset

#### TC-2.2.1: Valid email

```
POST {{base_url}}/users/password/reset/request/
```

```json
{
  "email": "admin@example.com"
}
```

**Expected:** `200 OK`

```json
{
  "detail": "If an account with that email exists, a reset link has been sent."
}
```

**Verify:** Check email/console output for uid + token.

---

#### TC-2.2.2: Nonexistent email (no enumeration)

```json
{
  "email": "doesnotexist@example.com"
}
```

**Expected:** `200 OK` — same response (no leak).

---

#### TC-2.2.3: Invalid email format

```json
{
  "email": "not-an-email"
}
```

**Expected:** `400 Bad Request` — validation error.

---

#### TC-2.2.4: Rate limit (3/min)

Send 4 requests rapidly.

**Expected:** 4th request returns `429 Too Many Requests`.

---

### P2.3 — `POST /users/password/reset/confirm/` — Confirm Reset

#### TC-2.3.1: Valid uid + token + password

```
POST {{base_url}}/users/password/reset/confirm/
```

```json
{
  "uid": "<uid_from_email>",
  "token": "<token_from_email>",
  "new_password": "newsecure123"
}
```

**Expected:** `200 OK`

```json
{
  "detail": "Password has been reset successfully."
}
```

**Verify:** Login with new password succeeds.

---

#### TC-2.3.2: Invalid token

```json
{
  "uid": "<valid_uid>",
  "token": "invalid-token-value",
  "new_password": "newsecure123"
}
```

**Expected:** `400 Bad Request`

```json
{
  "error": "Invalid or expired reset link."
}
```

---

#### TC-2.3.3: Short password

```json
{
  "uid": "<valid_uid>",
  "token": "<valid_token>",
  "new_password": "short"
}
```

**Expected:** `400 Bad Request` — validation error (min 8 characters).

---

### P2.4 — `POST /users/{id}/reset-password/` — Admin Password Reset

#### TC-2.4.1: Admin resets with custom password

```
POST {{base_url}}/users/{{user_id}}/reset-password/
Authorization: Bearer {{access_token}}
```

```json
{
  "new_password": "AdminSet123!"
}
```

**Expected:** `200 OK`

```json
{
  "user_id": "uuid",
  "username": "student_user",
  "password": "AdminSet123!",
  "generated": false
}
```

---

#### TC-2.4.2: Admin resets without password (auto-generated)

```json
{}
```

**Expected:** `200 OK`

```json
{
  "user_id": "uuid",
  "username": "student_user",
  "password": "<random_12_char>",
  "generated": true
}
```

---

#### TC-2.4.3: Nonexistent user

```
POST {{base_url}}/users/00000000-0000-0000-0000-000000000000/reset-password/
```

**Expected:** `400 Bad Request`

```json
{
  "error": "User not found or inactive."
}
```

---

#### TC-2.4.4: Unauthenticated

No `Authorization` header.

**Expected:** `401 Unauthorized`

---

## P3. Geographic Hierarchy

**Prefix:** `{{base_url}}/schools/public`

All endpoints are public (no auth). Responses are paginated (`count`, `next`, `previous`, `results`).

### P3.1 — `GET .../centrals/`

#### TC-3.1.1: List all

```
GET {{base_url}}/schools/public/centrals/
```

**Expected:** `200 OK`

```json
{
  "count": 1,
  "next": null,
  "previous": null,
  "results": [
    { "id": "uuid", "name": "Central Office" }
  ]
}
```

---

#### TC-3.1.2: Search by name

```
GET {{base_url}}/schools/public/centrals/?search=central
```

**Expected:** `200 OK` — filtered results.

---

### P3.2 — `GET .../regions/`

#### TC-3.2.1: Filter by central

```
GET {{base_url}}/schools/public/regions/?central={{central_id}}
```

**Expected:** `200 OK` — only regions under that central.

---

#### TC-3.2.2: No filter

```
GET {{base_url}}/schools/public/regions/
```

**Expected:** `200 OK` — all regions.

---

#### TC-3.2.3: Invalid UUID

```
GET {{base_url}}/schools/public/regions/?central=not-a-uuid
```

**Expected:** `200 OK` — empty results (filter silently returns nothing).

---

### P3.3 — `GET .../divisions/`

#### TC-3.3.1: Filter by region

```
GET {{base_url}}/schools/public/divisions/?region={{region_id}}
```

**Expected:** `200 OK` — divisions under that region.

---

### P3.4 — `GET .../districts/`

#### TC-3.4.1: Filter by division

```
GET {{base_url}}/schools/public/districts/?division={{division_id}}
```

**Expected:** `200 OK` — districts under that division.

---

### P3.5 — `GET .../schools/`

#### TC-3.5.1: Filter by district

```
GET {{base_url}}/schools/public/schools/?district={{district_id}}
```

**Expected:** `200 OK`

```json
{
  "results": [
    {
      "id": "uuid",
      "name": "Test National High School",
      "school_id": "302939",
      "is_registered": true
    }
  ]
}
```

**Verify:** `is_registered` field is present in every result.

---

#### TC-3.5.2: No filter

```
GET {{base_url}}/schools/public/schools/
```

**Expected:** `200 OK` — all schools.

---

## P4. Academic Programs

### P4.1 — `GET .../academic-programs/`

#### TC-4.1.1: Full tree

```
GET {{base_url}}/schools/public/academic-programs/
```

**Expected:** `200 OK` — recursive tree.

```json
[
  {
    "id": "uuid",
    "name": "K to 12 Basic Education",
    "code": "K12",
    "offering_type": "system",
    "is_leaf": false,
    "children": [
      {
        "id": "uuid",
        "name": "Elementary",
        "code": "ELEM",
        "offering_type": "program",
        "is_leaf": false,
        "children": [...]
      }
    ]
  }
]
```

**Verify:**
- Root nodes have `is_leaf: false`
- Leaf nodes (actual grade levels) have `is_leaf: true` and `children: []`

---

### P4.2 — `GET .../school-academic-programs/`

#### TC-4.2.1: With school filter

```
GET {{base_url}}/schools/public/school-academic-programs/?school={{school_id}}
```

**Expected:** `200 OK` — pruned tree (only branches containing offered programs).

**Verify:**
- Leaf nodes with `is_offered: true` are the programs the school offers
- Branches with no offered descendants are excluded

---

#### TC-4.2.2: Missing school param

```
GET {{base_url}}/schools/public/school-academic-programs/
```

**Expected:** `400 Bad Request`

```json
{
  "error": "school query param is required."
}
```

---

#### TC-4.2.3: Nonexistent school

```
GET {{base_url}}/schools/public/school-academic-programs/?school=00000000-0000-0000-0000-000000000000
```

**Expected:** `200 OK` — empty array `[]`.

---

## P5. Announcements

### P5.1 — `GET .../announcements/`

#### TC-5.1.1: With school filter (cascading visibility)

```
GET {{base_url}}/schools/public/announcements/?school={{school_id}}
```

**Expected:** `200 OK` — announcements scoped at the school level AND every ancestor (district, division, region, central).

```json
{
  "results": [
    {
      "id": "uuid",
      "title": "Welcome Back",
      "body": "...",
      "scope_level": "school",
      "scope_name": "Test School (302939)",
      "priority": "normal",
      "is_pinned": false,
      "author_name": "admin",
      "published_at": "2026-02-24T00:00:00Z",
      "expires_at": null,
      "created": "..."
    }
  ]
}
```

**Verify:**
- Pinned announcements appear first
- Only `published` status announcements returned
- Announcements from parent geo levels (region, central) are included

---

#### TC-5.1.2: Missing school param

```
GET {{base_url}}/schools/public/announcements/
```

**Expected:** `400 Bad Request`

```json
{
  "error": "Validation failed.",
  "extra": {
    "code": "VALIDATION_ERROR",
    "fields": { "school": ["This query parameter is required."] }
  }
}
```

---

#### TC-5.1.3: School with no announcements

**Expected:** `200 OK` — empty `results: []`.

---

## P6. Students

**Prefix:** `{{base_url}}/students`

### P6.1 — `POST /students/onboard/` — Single Student Onboard

**Content-Type:** `multipart/form-data`
**Auth:** Required

#### TC-6.1.1: Minimal fields

```
POST {{base_url}}/students/onboard/
Authorization: Bearer {{access_token}}
Content-Type: multipart/form-data

school_id: {{school_id}}
first_name: Juan
last_name: Dela Cruz
lrn: 120345678901
```

**Expected:** `201 Created`

```json
{
  "created_count": 1,
  "student_ids": ["uuid"]
}
```

---

#### TC-6.1.2: All fields + photo + guardian

```
POST {{base_url}}/students/onboard/
Content-Type: multipart/form-data

school_id: {{school_id}}
first_name: Maria
last_name: Santos
lrn: 120345678902
middle_name: Reyes
suffix: Jr
sex: Female
date_of_birth: 2014-03-15
place_of_birth: Quezon City
phone_number: 09171234567
mother_tongue: Filipino
belongs_to_ip: false
is_4ps_beneficiary: false
is_pwd: false
is_regular: true
photo: [attach image file]
guardian_first_name: Pedro
guardian_last_name: Santos
guardian_contact_number: 09181234567
guardian_relationship: father
```

**Expected:** `201 Created`

---

#### TC-6.1.3: Duplicate LRN

Send the same LRN as TC-6.1.1 again.

**Expected:** `400 Bad Request` — integrity/validation error.

---

#### TC-6.1.4: Invalid school_id

```
school_id: 00000000-0000-0000-0000-000000000000
```

**Expected:** `404 Not Found`

```json
{
  "error": "School '00000000-...' not found.",
  "extra": { "code": "NOT_FOUND" }
}
```

---

#### TC-6.1.5: Unauthenticated

No `Authorization` header.

**Expected:** `401 Unauthorized`

---

### P6.2 — `POST /students/onboard/batch/` — Batch Onboard

#### TC-6.2.1: JSON payload

```
POST {{base_url}}/students/onboard/batch/
Authorization: Bearer {{access_token}}
Content-Type: application/json
```

```json
{
  "school_id": "{{school_id}}",
  "students": [
    {
      "first_name": "Student",
      "last_name": "One",
      "lrn": "120000000001"
    },
    {
      "first_name": "Student",
      "last_name": "Two",
      "lrn": "120000000002",
      "sex": "Male",
      "guardian_first_name": "Parent",
      "guardian_last_name": "Two",
      "guardian_relationship": "mother"
    }
  ]
}
```

**Expected:** `201 Created`

```json
{
  "created_count": 2,
  "student_ids": ["uuid1", "uuid2"]
}
```

---

#### TC-6.2.2: CSV upload

```
POST {{base_url}}/students/onboard/batch/
Authorization: Bearer {{access_token}}
Content-Type: multipart/form-data

school_id: {{school_id}}
file: [attach students.csv]
```

**CSV format:**

```csv
lrn,last_name,first_name,middle_name,sex,guardian_last_name,guardian_first_name,guardian_contact,relationship
120000000010,Cruz,Ana,Santos,Female,Cruz,Pedro,09171234567,father
120000000011,Reyes,Ben,,Male,Reyes,Maria,09181234567,mother
```

**Expected:** `201 Created`

---

#### TC-6.2.3: CSV missing required columns

CSV without `lrn` column.

**Expected:** `400 Bad Request`

```json
{
  "error": "Missing required CSV columns: lrn",
  "extra": { "code": "CSV_PARSE_ERROR" }
}
```

---

#### TC-6.2.4: CSV over 5 MB

**Expected:** `400 Bad Request` — "CSV file must be under 5 MB."

---

#### TC-6.2.5: Empty students array

```json
{
  "school_id": "{{school_id}}",
  "students": []
}
```

**Expected:** `400 Bad Request` — validation error (min_length=1).

---

### P6.3 — `POST /students/onboard/batch-photos/` — Batch Photo Upload

**Content-Type:** `multipart/form-data`
**Auth:** Required

#### TC-6.3.1: Valid LRN-named photos

```
POST {{base_url}}/students/onboard/batch-photos/
Authorization: Bearer {{access_token}}
Content-Type: multipart/form-data

photos: 120345678901.jpg
photos: 120345678902.jpg
```

**Expected:** `200 OK`

```json
{
  "matched": 2,
  "not_found": [],
  "errors": []
}
```

---

#### TC-6.3.2: Non-12-digit filename

Upload `INVALID.jpg`.

**Expected:** `200 OK` — error listed.

```json
{
  "matched": 0,
  "not_found": [],
  "errors": ["INVALID.jpg: filename must be a 12-digit LRN"]
}
```

---

#### TC-6.3.3: LRN not found in DB

Upload `999999999999.jpg` (no matching student).

**Expected:** `200 OK`

```json
{
  "matched": 0,
  "not_found": ["999999999999"],
  "errors": []
}
```

---

#### TC-6.3.4: No files

Send empty request (no `photos` field).

**Expected:** `400 Bad Request`

```json
{
  "error": "No files uploaded. Send photos under the 'photos' field."
}
```

---

### P6.4 — `GET /students/disabilities/` — Disability Tree

#### TC-6.4.1: Returns tree

```
GET {{base_url}}/students/disabilities/
```

**Expected:** `200 OK` — tree with parent categories and children.

```json
[
  {
    "id": "uuid",
    "name": "Visual Impairment",
    "children": [
      { "id": "uuid", "name": "Low Vision" },
      { "id": "uuid", "name": "Blind" }
    ]
  }
]
```

---

### P6.5 — `GET /students/modalities/` — Modality List

#### TC-6.5.1: Returns flat list

```
GET {{base_url}}/students/modalities/
```

**Expected:** `200 OK`

```json
[
  { "id": "uuid", "name": "Face-to-Face" },
  { "id": "uuid", "name": "Blended Learning" }
]
```

---

## P7. Enrollment — Public

**Prefix:** `{{base_url}}/enrollment`

### P7.1 — `GET /enrollment/settings/`

#### TC-7.1.1: With school filter

```
GET {{base_url}}/enrollment/settings/?school={{school_id}}
```

**Expected:** `200 OK` — list of enrollment settings per program.

```json
{
  "results": [
    {
      "id": "uuid",
      "program": "uuid",
      "program_name": "Grade 7",
      "enrollment_mode": "registrar_assign",
      "max_section_choices": 3,
      "registrar_override_enabled": true,
      "auto_balance_strategy": "none",
      "requires_exam": false
    }
  ]
}
```

---

#### TC-7.1.2: No settings for school

Use a school with no SchoolEnrollmentSettings records.

**Expected:** `200 OK` — empty `results: []`.

---

### P7.2 — `GET /enrollment/section-offerings/`

#### TC-7.2.1: With school + program

```
GET {{base_url}}/enrollment/section-offerings/?school={{school_id}}&program={{program_id}}
```

**Expected:** `200 OK`

```json
{
  "results": [
    {
      "id": "uuid",
      "section_name": "Section A",
      "program": "uuid",
      "program_code": "G7",
      "program_name": "Grade 7",
      "specialization_id": null,
      "specialization_name": null,
      "max_students": 40,
      "current_students": 12
    }
  ]
}
```

---

#### TC-7.2.2: Missing school param

```
GET {{base_url}}/enrollment/section-offerings/
```

**Expected:** `400 Bad Request`

---

### P7.3 — `POST /enrollment/submissions/` — Submit BEF

#### TC-7.3.1: Valid submission

```
POST {{base_url}}/enrollment/submissions/
```

```json
{
  "school": "{{school_id}}",
  "program": "{{program_id}}",
  "student_data": {
    "first_name": "Juan",
    "last_name": "Dela Cruz",
    "lrn": "120345678901",
    "middle_name": "Santos",
    "date_of_birth": "2014-03-15",
    "sex": "Male",
    "mother_tongue": "Filipino",
    "father": {
      "first_name": "Pedro",
      "last_name": "Dela Cruz",
      "contact_number": "09171234567"
    }
  }
}
```

**Expected:** `201 Created`

```json
{
  "id": "uuid",
  "school": "uuid",
  "school_academic_term": "uuid",
  "academic_program": "uuid",
  "section_preferences": [],
  "status": "pending",
  "created": "..."
}
```

**Save:** `id` → `{{submission_id}}`

---

#### TC-7.3.2: Missing first_name/last_name

```json
{
  "school": "{{school_id}}",
  "program": "{{program_id}}",
  "student_data": {
    "lrn": "120345678901"
  }
}
```

**Expected:** `400 Bad Request` — "student_data.first_name is required."

---

#### TC-7.3.3: School not found

```json
{
  "school": "00000000-0000-0000-0000-000000000000",
  "program": "{{program_id}}",
  "student_data": { "first_name": "A", "last_name": "B" }
}
```

**Expected:** `400 Bad Request` — "School not found."

---

#### TC-7.3.4: Enrollment window closed

Use a school whose `SchoolAcademicTerm.enrollment_end_date` is in the past.

**Expected:** `400 Bad Request` — "Enrollment is not open for this school."

---

#### TC-7.3.5: Section preferences (STUDENT_PRIORITY mode)

Pre-condition: School has `enrollment_mode=student_priority`, `max_section_choices=3`.

```json
{
  "school": "{{school_id}}",
  "program": "{{program_id}}",
  "student_data": { "first_name": "A", "last_name": "B" },
  "section_preferences": [
    "{{section_offering_id_1}}",
    "{{section_offering_id_2}}"
  ]
}
```

**Expected:** `201 Created` — `section_preferences` populated.

---

#### TC-7.3.6: Too many preferences

Send more preferences than `max_section_choices`.

**Expected:** `400 Bad Request` — "Too many section preferences (max N)."

---

#### TC-7.3.7: Duplicate preferences

```json
{
  "section_preferences": ["{{same_uuid}}", "{{same_uuid}}"]
}
```

**Expected:** `400 Bad Request` — "Duplicate section preferences are not allowed."

---

### P7.4 — `GET /enrollment/submissions/{id}/status/`

#### TC-7.4.1: Valid submission

```
GET {{base_url}}/enrollment/submissions/{{submission_id}}/status/
```

**Expected:** `200 OK`

```json
{
  "id": "uuid",
  "status": "pending",
  "remarks": "",
  "reviewed_at": null,
  "created": "..."
}
```

---

#### TC-7.4.2: Nonexistent ID

```
GET {{base_url}}/enrollment/submissions/00000000-0000-0000-0000-000000000000/status/
```

**Expected:** `404 Not Found`

---

## P8. Enrollment — Admin

**Auth:** Required (JWT, scoped to school)

### P8.1 — `GET /enrollment/submissions/` — List Submissions

#### TC-8.1.1: List all (scoped)

```
GET {{base_url}}/enrollment/submissions/
Authorization: Bearer {{access_token}}
```

**Expected:** `200 OK` — only submissions for the registrar's school.

---

#### TC-8.1.2: Filter by status

```
GET {{base_url}}/enrollment/submissions/?status=pending
```

**Expected:** `200 OK` — only pending submissions.

---

#### TC-8.1.3: Combined filters

```
GET {{base_url}}/enrollment/submissions/?status=pending&term={{term_id}}&program={{program_id}}
```

**Expected:** `200 OK` — intersection of all filters.

---

#### TC-8.1.4: Unauthenticated

No `Authorization` header.

**Expected:** `401 Unauthorized`

---

### P8.2 — `GET /enrollment/submissions/{id}/` — Submission Detail

#### TC-8.2.1: Valid ID

```
GET {{base_url}}/enrollment/submissions/{{submission_id}}/
Authorization: Bearer {{access_token}}
```

**Expected:** `200 OK` — full submission detail including `student_data` JSON and `section_preference_details`.

---

### P8.3 — `PATCH /enrollment/submissions/{id}/review/` — Review Submission

#### TC-8.3.1: Admit with section offering

```
PATCH {{base_url}}/enrollment/submissions/{{submission_id}}/review/
Authorization: Bearer {{access_token}}
```

```json
{
  "status": "admitted",
  "section_offering": "{{section_offering_id}}",
  "remarks": "Approved for enrollment"
}
```

**Expected:** `200 OK`

**Verify:**
- `status` changed to `"admitted"`
- `admitted_student` is set (UUID of created Student)
- `reviewed_at` is set
- New Student, User, Guardian records exist in DB
- StudentBridgeSync row created with `status=pending`

---

#### TC-8.3.2: Admit without section_offering

```json
{
  "status": "admitted",
  "remarks": "test"
}
```

**Expected:** `400 Bad Request`

```json
{
  "error": "Validation failed.",
  "extra": {
    "code": "VALIDATION_ERROR",
    "fields": { "section_offering": ["Required when admitting a student."] }
  }
}
```

---

#### TC-8.3.3: Reject with remarks

```json
{
  "status": "rejected",
  "remarks": "Incomplete documents"
}
```

**Expected:** `200 OK` — `status` changed to `"rejected"`, `expires_at` set.

---

#### TC-8.3.4: Waitlist

```json
{
  "status": "waitlisted",
  "remarks": "Section full, waitlisted for next opening"
}
```

**Expected:** `200 OK` — `status` changed to `"waitlisted"`.

---

#### TC-8.3.5: Admit with duplicate LRN

Submit and admit a BEF with the same LRN as an existing student.

**Expected:** `400 Bad Request` — "A student with LRN ... already exists."

---

#### TC-8.3.6: Admit with nonexistent section_offering

```json
{
  "status": "admitted",
  "section_offering": "00000000-0000-0000-0000-000000000000"
}
```

**Expected:** `400 Bad Request` — "SectionOffering not found."

---

## P9. Academic Year Rollover

### P9.1 — `POST /enrollment/rollover/`

#### TC-9.1.1: Dry run (preview)

```
POST {{base_url}}/enrollment/rollover/
Authorization: Bearer {{access_token}}
```

```json
{
  "school": "{{school_id}}",
  "academic_term": "{{academic_term_id}}",
  "start_date": "2026-06-01",
  "end_date": "2027-03-31",
  "enrollment_start_date": "2026-04-01",
  "enrollment_end_date": "2026-06-30",
  "source_term": "{{source_term_id}}",
  "copy_sections": true,
  "sub_terms": [
    {
      "academic_term": "{{q1_term_id}}",
      "start_date": "2026-06-01",
      "end_date": "2026-08-15"
    },
    {
      "academic_term": "{{q2_term_id}}",
      "start_date": "2026-08-16",
      "end_date": "2026-10-31"
    }
  ],
  "dry_run": true
}
```

**Expected:** `200 OK`

```json
{
  "term_preview": {
    "academic_term": "School Year",
    "start_date": "2026-06-01",
    "end_date": "2027-03-31",
    "enrollment_start_date": "2026-04-01",
    "enrollment_end_date": "2026-06-30"
  },
  "sections": {
    "to_create": [...],
    "carried_over": [...],
    "needs_attention": [...]
  },
  "students": {
    "eligible": [...],
    "retained": [...],
    "incomplete": [...]
  }
}
```

**Verify:** No records created in the database.

---

#### TC-9.1.2: Live run

Same payload but `"dry_run": false`.

**Expected:** `200 OK` — response includes `term_created.id` and `sections.created_count`.

**Verify:**
- New `SchoolAcademicTerm` created
- Sub-terms created as children
- `SectionOffering` records created per plan

---

#### TC-9.1.3: Duplicate term

Run TC-9.1.2 again with same parameters.

**Expected:** `400 Bad Request` — "A SchoolAcademicTerm for ... starting ... already exists."

---

#### TC-9.1.4: Invalid date ranges

```json
{
  "start_date": "2027-06-01",
  "end_date": "2026-03-31"
}
```

**Expected:** `400 Bad Request` — "Must be after start_date."

---

#### TC-9.1.5: Nonexistent school

```json
{
  "school": "00000000-0000-0000-0000-000000000000"
}
```

**Expected:** `400 Bad Request` — "School ... not found."

---

## P10. Realtime

### P10.1 — `POST /realtime/telegram/webhook/{school_id}/`

#### TC-10.1.1: Valid Telegram update

```
POST {{base_url}}/realtime/telegram/webhook/{{school_id}}/
```

```json
{
  "update_id": 123456,
  "message": {
    "message_id": 1,
    "from": { "id": 12345, "first_name": "Test" },
    "chat": { "id": 12345, "type": "private" },
    "date": 1708000000,
    "text": "/start"
  }
}
```

**Expected:** `200 OK`

---

#### TC-10.1.2: Invalid school_id

```
POST {{base_url}}/realtime/telegram/webhook/00000000-0000-0000-0000-000000000000/
```

**Expected:** `400` or `404` — school not found / no messaging config.

---

### P10.2 — `GET /realtime/telegram/webhook/{school_id}/`

#### TC-10.2.1: Health check

```
GET {{base_url}}/realtime/telegram/webhook/{{school_id}}/
```

**Expected:** `200 OK`

---

## P11. End-to-End Flows

### E1. Full Enrollment Cycle

| Step | Action | Verify |
|---|---|---|
| 1 | `POST /enrollment/submissions/` — submit BEF | `201`, status=pending |
| 2 | `GET /enrollment/submissions/` — list (as registrar) | Submission appears |
| 3 | `GET /enrollment/submissions/{id}/` — detail | Full student_data visible |
| 4 | `PATCH /enrollment/submissions/{id}/review/` — admit | `200`, status=admitted |
| 5 | `GET /enrollment/submissions/{id}/status/` — public check | status=admitted |
| 6 | `POST /students/onboard/batch-photos/` — upload photo | matched=1 |
| 7 | Verify in DB/admin: StudentBridgeSync status=pending | Bridge waiting for sync |

---

### E2. Student Onboarding via CSV

| Step | Action | Verify |
|---|---|---|
| 1 | `POST /students/onboard/batch/` — upload CSV | `201`, created_count matches rows |
| 2 | Check admin: Students created with auto-generated usernames | Usernames follow `first.last.lrn` pattern |
| 3 | `POST /students/onboard/batch-photos/` — upload photos | matched count > 0 |
| 4 | Check admin: User.profile_image set | Photos stored in uploads/profile_pictures/ |
| 5 | Check admin: StudentBridgeSync rows exist, status=pending | Ready for bridge sync |

---

### E3. Password Reset

| Step | Action | Verify |
|---|---|---|
| 1 | `POST /users/password/reset/request/` with valid email | `200`, check console/email for uid+token |
| 2 | `POST /users/password/reset/confirm/` with uid+token+new_password | `200`, "Password has been reset" |
| 3 | `POST /auth/token/` — login with NEW password | `200`, tokens returned |
| 4 | `POST /auth/token/` — login with OLD password | `401`, fails |

---

### E4. Academic Year Rollover

| Step | Action | Verify |
|---|---|---|
| 1 | `POST /enrollment/rollover/` with `dry_run: true` | `200`, preview with sections plan |
| 2 | Review `sections.to_create` and `sections.needs_attention` | Lifecycle rules applied correctly |
| 3 | `POST /enrollment/rollover/` with `dry_run: false` | `200`, `term_created.id` present |
| 4 | Check admin: SchoolAcademicTerm created | Dates match request |
| 5 | Check admin: Sub-terms created as children | Linked to parent term |
| 6 | Check admin: SectionOfferings created | Programs match plan |

---

### E5. Login with Email

| Step | Action | Verify |
|---|---|---|
| 1 | Set email on a user via admin (e.g. `testuser@example.com`) | Email saved |
| 2 | `POST /auth/token/` with `{"username": "testuser@example.com", "password": "..."}` | `200`, tokens returned |
| 3 | `GET /users/me/` with the token | Correct user returned |

---

## Appendix: Error Response Format

All API errors follow this consistent envelope:

```json
{
  "error": "Human-readable error message",
  "extra": {
    "code": "MACHINE_CODE",
    "fields": {
      "field_name": ["Error detail"]
    }
  }
}
```

Common codes:
- `VALIDATION_ERROR` — input validation failed (400)
- `AUTHENTICATION_FAILED` — wrong credentials (401)
- `NOT_AUTHENTICATED` — missing auth header (401)
- `NOT_FOUND` — resource not found (404)
- `THROTTLED` — rate limited (429)
- `INTERNAL_ERROR` — unhandled server error (500)
