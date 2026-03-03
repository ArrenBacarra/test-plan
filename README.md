# SMIS V2 — Cross-School Data Integrity & Edge Case Test Plan

**Purpose:** Catch "weird" scenarios where data from School A leaks into School B — wrong section offerings, wrong terms, wrong scope, etc. These tests target the **seams** between school-scoped data.

**Base URL:** `{{base_url}}` (e.g. `http://localhost:8000/api/v1`)
**Auth:** Bearer JWT — tests use two accounts: one scoped to **School A**, one to **School B**.

---

## Table of Contents

1. [Setup](#setup)
2. [X1. Enrollment — Cross-School Section Offering](#x1-enrollment--cross-school-section-offering)
3. [X2. Enrollment — Cross-School Term Mismatch](#x2-enrollment--cross-school-term-mismatch)
4. [X3. Enrollment — Program Not Offered by School](#x3-enrollment--program-not-offered-by-school)
5. [X4. Admission — Section Offering from Wrong School](#x4-admission--section-offering-from-wrong-school)
6. [X5. Admission — Section Offering from Wrong Term](#x5-admission--section-offering-from-wrong-term)
7. [X6. Admission — Section Offering Wrong Program](#x6-admission--section-offering-wrong-program)
8. [X7. Student Onboarding — Cross-School Scope Bypass](#x7-student-onboarding--cross-school-scope-bypass)
9. [X8. Rollover — Source Term from Different School](#x8-rollover--source-term-from-different-school)
10. [X9. Rollover — Target School Outside User Scope](#x9-rollover--target-school-outside-user-scope)
11. [X10. Submission — School A User Reviews School B Submission](#x10-submission--school-a-user-reviews-school-b-submission)
12. [X11. Public Endpoints — Forged School IDs](#x11-public-endpoints--forged-school-ids)
13. [X12. Section Offerings Dropdown — Program from Another School](#x12-section-offerings-dropdown--program-from-another-school)
14. [X13. Enrollment Submission — Section Prefs from Another School](#x13-enrollment-submission--section-prefs-from-another-school)
15. [X14. Bridge/Realtime — Event School Mismatch](#x14-bridgerealtime--event-school-mismatch)
16. [X15. Batch Onboard — Mixed-School LRN Collision](#x15-batch-onboard--mixed-school-lrn-collision)
17. [X16. Admin — Cross-School Object Save](#x16-admin--cross-school-object-save)
18. [Severity Matrix](#severity-matrix)
19. [Postman Automation Scripts](#postman-automation-scripts)

---

## Setup

### Required Test Data (two schools, two users)

| Entity | Alias | Notes |
|---|---|---|
| School A | `{{school_a_id}}` | Registered, has programs, enrollment open, sections created |
| School B | `{{school_b_id}}` | Registered, has programs, enrollment open, sections created |
| Registrar A | `{{token_a}}` | Staff scoped to School A |
| Registrar B | `{{token_b}}` | Staff scoped to School B |
| Superuser | `{{token_super}}` | Global admin (no school scope) |
| Section Offering A | `{{so_a_id}}` | Belongs to School A's term + section |
| Section Offering B | `{{so_b_id}}` | Belongs to School B's term + section |
| Term A | `{{term_a_id}}` | `SchoolAcademicTerm` for School A |
| Term B | `{{term_b_id}}` | `SchoolAcademicTerm` for School B |
| Program A | `{{program_a_id}}` | Offered by School A (not School B) |
| Program B | `{{program_b_id}}` | Offered by School B (not School A) |

### Environment Variables (Postman)

Add these manually alongside the existing `SMIS Local` environment:

| Variable | Description |
|---|---|
| `school_a_id` | UUID of School A |
| `school_b_id` | UUID of School B |
| `token_a` | Access token for Registrar A |
| `token_b` | Access token for Registrar B |
| `token_super` | Access token for superuser |
| `so_a_id` | SectionOffering UUID from School A |
| `so_b_id` | SectionOffering UUID from School B |
| `term_a_id` | SchoolAcademicTerm UUID for School A |
| `term_b_id` | SchoolAcademicTerm UUID for School B |
| `program_a_id` | AcademicProgram offered only at School A |
| `program_b_id` | AcademicProgram offered only at School B |

### How to Populate

1. Login as Registrar A → save `{{token_a}}`
2. Login as Registrar B → save `{{token_b}}`
3. Login as superuser → save `{{token_super}}`
4. Run the dropdown flow (P3.6) for each school to get IDs
5. Grab section offering IDs from `GET /enrollment/section-offerings/?school=...`

---

## X1. Enrollment — Cross-School Section Offering

> **Scenario:** A public enrollment submission sends School A's ID but includes a section preference from School B.

#### TC-X1.1: Section preference belongs to wrong school

```
POST {{base_url}}/enrollment/submissions/
```

```json
{
  "school": "{{school_a_id}}",
  "program": "{{program_a_id}}",
  "student_data": {
    "first_name": "Cross",
    "last_name": "School",
    "lrn": "120099990001"
  },
  "section_preferences": ["{{so_b_id}}"]
}
```

**Expected:** `400 Bad Request` — section preference not valid for this school/term/program. The service filters `SectionOffering.objects.filter(section__school=school, ...)`, so School B's offering should be rejected.

**What to verify:**
- The `so_b_id` is NOT accepted as a valid preference
- Response explains the preference is invalid
- No submission is created

**Why it matters:** If this passes (201), a student at School A has a preference pointing to School B's section — corrupted data.

---

## X2. Enrollment — Cross-School Term Mismatch

> **Scenario:** The `EnrollmentSubmission` model has a `clean()` that validates `school_academic_term.school == school`. But `clean()` is only called on individual saves. Does `create_enrollment_submission` enforce this?

#### TC-X2.1: Term auto-resolved from school (no direct injection)

The `create_enrollment_submission` service resolves the term via `enrollment_open_term_get(school=school)`, so the term always matches the school. **This is safe by design.**

**Verify:** No way to pass a `term` directly in the public submission API. The term is auto-resolved.

**Status:** PASS (by architecture). No manual test needed — just confirm the API schema doesn't accept a `term` field.

---

## X3. Enrollment — Program Not Offered by School

> **Scenario:** Submit an enrollment for School A but use a program that only School B offers.

#### TC-X3.1: Program not offered at the target school

```
POST {{base_url}}/enrollment/submissions/
```

```json
{
  "school": "{{school_a_id}}",
  "program": "{{program_b_id}}",
  "student_data": {
    "first_name": "Wrong",
    "last_name": "Program",
    "lrn": "120099990002"
  }
}
```

**Expected (ideal):** `400 Bad Request` — "Program is not offered by this school."

**Current behavior (may differ):** The service fetches the program by ID without checking `SchoolAcademicProgram`. It may succeed, creating a submission for a program the school doesn't actually offer.

**Severity:** **HIGH** if it creates the submission — the registrar will see a submission for a program their school doesn't offer, and admission will create a `StudentHistory` referencing that program.

---

## X4. Admission — Section Offering from Wrong School

> **This is the exact bug you found.** A registrar admits a student using a section offering that belongs to a different school.

#### TC-X4.1: Admit with School B's section offering on School A's submission

Pre-condition: Create a pending submission at School A.

```
PATCH {{base_url}}/enrollment/submissions/{{submission_a_id}}/review/
Authorization: Bearer {{token_a}}
```

```json
{
  "status": "admitted",
  "section_offering": "{{so_b_id}}",
  "remarks": "Cross-school section test"
}
```

**Expected (ideal):** `400 Bad Request` — "Section offering does not belong to this school."

**Current behavior (may differ):** The `admit_enrollment_submission` service only checks `SectionOffering.objects.get(pk=section_offering_id)` — no school cross-check. It may succeed, creating:
- A `Student` at School A
- A `StudentHistory` pointing to School B's `SectionOffering`
- An incremented `number_of_students` counter on School B's section

**Severity:** **CRITICAL** — This corrupts student records across schools. School B's section count inflates. Student appears in wrong school's class roster.

**Verify if it passes (201/200):**
- `StudentHistory.section_offering.section.school` ≠ `student.school` → data corruption
- School B's section `number_of_students` was incremented for a student not actually in it

---

#### TC-X4.2: Admit with section offering from correct school but wrong term

Pre-condition: School A has two terms (e.g. SY 2025-2026 and SY 2024-2025). Create a section offering for the old term.

```json
{
  "status": "admitted",
  "section_offering": "{{so_a_old_term_id}}",
  "remarks": "Wrong term section test"
}
```

**Expected (ideal):** `400 Bad Request` — "Section offering is not for the submission's academic term."

**Current behavior:** No term cross-check in `admit_enrollment_submission`.

**Severity:** **HIGH** — Student gets placed in a section from last year's term.

---

## X5. Admission — Section Offering from Wrong Term

> Covered in TC-X4.2 above. Separated for clarity.

#### TC-X5.1: Term mismatch — section offering from previous academic year

Same as TC-X4.2. The `section_offering.school_academic_term` does not match `submission.school_academic_term`.

**Expected:** `400 Bad Request`

---

## X6. Admission — Section Offering Wrong Program

> **Scenario:** Admit a student from a Grade 7 submission into a Grade 8 section offering.

#### TC-X6.1: Program mismatch between submission and section offering

Pre-condition:
- Submission is for `program_id` = Grade 7
- `section_offering` is for Grade 8 (same school, same term)

```json
{
  "status": "admitted",
  "section_offering": "{{so_grade8_id}}",
  "remarks": "Program mismatch test"
}
```

**Expected (ideal):** `400 Bad Request` — "Section offering program does not match submission program."

**Current behavior:** No program cross-check. Student may be admitted into the wrong grade.

**Severity:** **HIGH** — Grade 7 student ends up in Grade 8's class roster.

---

## X7. Student Onboarding — Cross-School Scope Bypass

> **Scenario:** Registrar A (scoped to School A) onboards students to School B by sending School B's ID in the request body.

#### TC-X7.1: Registrar A onboards to School B (JSON)

```
POST {{base_url}}/students/onboard/batch/
Authorization: Bearer {{token_a}}
Content-Type: application/json
```

```json
{
  "school_id": "{{school_b_id}}",
  "students": [
    {
      "first_name": "Rogue",
      "last_name": "Student",
      "lrn": "120099990003"
    }
  ]
}
```

**Expected (ideal):** `403 Forbidden` — "Cannot onboard students outside your school scope."

**Current behavior (may differ):** The view uses `permissions.IsAuthenticated` only and fetches the school from the request body. Serializer TODOs say `school_id` should come from scope. This likely succeeds.

**Severity:** **CRITICAL** — Any authenticated user can create student accounts at any school.

---

#### TC-X7.2: Registrar A onboards to School B (single + photo)

```
POST {{base_url}}/students/onboard/
Authorization: Bearer {{token_a}}
Content-Type: multipart/form-data

school_id: {{school_b_id}}
first_name: Rogue
last_name: Single
lrn: 120099990004
```

**Expected (ideal):** `403 Forbidden`

---

#### TC-X7.3: Registrar A onboards to School B (CSV)

```
POST {{base_url}}/students/onboard/batch/
Authorization: Bearer {{token_a}}
Content-Type: multipart/form-data

school_id: {{school_b_id}}
file: [CSV with students]
```

**Expected (ideal):** `403 Forbidden`

---

#### TC-X7.4: Superuser onboards to any school

```
POST {{base_url}}/students/onboard/batch/
Authorization: Bearer {{token_super}}
Content-Type: application/json
```

```json
{
  "school_id": "{{school_b_id}}",
  "students": [
    { "first_name": "Super", "last_name": "Admin", "lrn": "120099990005" }
  ]
}
```

**Expected:** `201 Created` — superusers/global admins should be allowed to operate on any school.

---

## X8. Rollover — Source Term from Different School

> **Scenario:** Rollover for School A but `source_term` is School B's term. Copies School B's sections into School A.

#### TC-X8.1: Copy sections from School B's term into School A

```
POST {{base_url}}/enrollment/rollover/
Authorization: Bearer {{token_a}}
```

```json
{
  "school": "{{school_a_id}}",
  "source_term": "{{term_b_id}}",
  "academic_term": "{{academic_term_id}}",
  "start_date": "2027-06-01",
  "end_date": "2028-03-31",
  "enrollment_start_date": "2027-04-01",
  "enrollment_end_date": "2027-06-30",
  "copy_sections": true,
  "dry_run": true
}
```

**Expected (ideal):** `400 Bad Request` — "Source term belongs to a different school."

**Current behavior (may differ):** The service filters `SectionOffering.objects.filter(school_academic_term_id=source_term_id)` without checking `source_term.school == school`. School B's sections may appear in the plan.

**Severity:** **HIGH** — School A ends up with sections copied from School B, referencing School B's Section objects.

**Verify if it passes:**
- `sections.to_create` contains sections that belong to School B
- If `dry_run: false`, actual `SectionOffering` rows created with School B's `Section` FKs at School A's new term

---

## X9. Rollover — Target School Outside User Scope

> **Scenario:** Registrar A triggers a rollover for School B.

#### TC-X9.1: Registrar A creates term at School B

```
POST {{base_url}}/enrollment/rollover/
Authorization: Bearer {{token_a}}
```

```json
{
  "school": "{{school_b_id}}",
  "academic_term": "{{academic_term_id}}",
  "start_date": "2027-06-01",
  "end_date": "2028-03-31",
  "enrollment_start_date": "2027-04-01",
  "enrollment_end_date": "2027-06-30",
  "copy_sections": false,
  "dry_run": false
}
```

**Expected (ideal):** `403 Forbidden` — "School is outside your scope."

**Current behavior (may differ):** `RolloverApi` extends `AuthenticatedBaseAPIView` (no scope check). This likely succeeds.

**Severity:** **CRITICAL** — Any authenticated user can create academic terms at any school.

---

## X10. Submission — School A User Reviews School B Submission

> **Scenario:** Registrar A tries to review/admit a submission that belongs to School B.

#### TC-X10.1: Registrar A patches School B's submission

```
PATCH {{base_url}}/enrollment/submissions/{{submission_b_id}}/review/
Authorization: Bearer {{token_a}}
```

```json
{
  "status": "admitted",
  "section_offering": "{{so_b_id}}",
  "remarks": "Cross-school review"
}
```

**Expected:** `404 Not Found` — the `ScopedBaseViewSet` scopes the queryset to Registrar A's school, so School B's submission isn't visible.

**This should already be safe** due to `SchoolHierarchyScopedQuerysetMixin`. Verify the 404.

---

#### TC-X10.2: Registrar A lists School B's submissions

```
GET {{base_url}}/enrollment/submissions/?school={{school_b_id}}
Authorization: Bearer {{token_a}}
```

**Expected:** `200 OK` with empty results (scoped queryset ignores the `school` filter param if outside scope), OR results only from School A.

---

## X11. Public Endpoints — Forged School IDs

> **Scenario:** Public endpoints that accept `school` as query param — can you pass a UUID that doesn't exist or is unregistered?

#### TC-X11.1: Nonexistent school in enrollment settings

```
GET {{base_url}}/enrollment/settings/?school=00000000-0000-0000-0000-000000000000
```

**Expected:** `404 Not Found` (registration gate: `get_registered_school_or_deny`)

---

#### TC-X11.2: SQL injection attempt in school param

```
GET {{base_url}}/enrollment/settings/?school=1' OR '1'='1
```

**Expected:** `400 Bad Request` or `404` — Django's UUID field rejects non-UUID strings.

---

#### TC-X11.3: School A's ID on School B's announcements endpoint

```
GET {{base_url}}/schools/public/announcements/?school={{school_a_id}}
```

**Expected:** `200 OK` — announcements for School A. This is **intended behavior** (public endpoint), not a vulnerability. Just verify it only returns School A's announcements + ancestors.

---

## X12. Section Offerings Dropdown — Program from Another School

> **Scenario:** Call the section offerings dropdown for School A but filter by a program only offered at School B.

#### TC-X12.1: Program filter from wrong school

```
GET {{base_url}}/enrollment/section-offerings/?school={{school_a_id}}&program={{program_b_id}}
```

**Expected:** `200 OK` with empty results — the query filters by `section__school=school_a` AND `academic_program_id=program_b`. Since School A has no section offerings for program B, results should be empty.

**Verify:** Response is `[]` or `{"results": []}`. No data from School B leaks.

---

## X13. Enrollment Submission — Section Prefs from Another School

> Expanded version of X1 — multiple preferences, mixed schools.

#### TC-X13.1: Mixed preferences (one valid, one from wrong school)

```json
{
  "school": "{{school_a_id}}",
  "program": "{{program_a_id}}",
  "student_data": {
    "first_name": "Mixed",
    "last_name": "Prefs",
    "lrn": "120099990010"
  },
  "section_preferences": ["{{so_a_id}}", "{{so_b_id}}"]
}
```

**Expected:** `400 Bad Request` — at least one preference is invalid. The service validates ALL preferences against `section__school=school`, so `so_b_id` fails.

**Verify:** The entire submission is rejected (not partially created).

---

#### TC-X13.2: All preferences from wrong school

```json
{
  "section_preferences": ["{{so_b_id}}"]
}
```

**Expected:** `400 Bad Request` — "Some section preferences are not valid."

---

## X14. Bridge/Realtime — Event School Mismatch

> **Scenario:** A bridge event arrives with `school_id` of School A but the student resolved from the event belongs to School B.

#### TC-X14.1: Telegram webhook with mismatched school

```
POST {{base_url}}/realtime/telegram/webhook/{{school_a_id}}/
```

```json
{
  "update_id": 999999,
  "message": {
    "message_id": 1,
    "from": { "id": 99999, "first_name": "Fake" },
    "chat": { "id": 99999, "type": "private" },
    "date": 1708000000,
    "text": "/start"
  }
}
```

**Expected:** `200 OK` — the webhook accepts the payload but the task should resolve the student's actual school. If no student is found for this chat, no notification is sent.

**Note:** This is harder to test via Postman since it depends on Telegram chat binding. Flag as a **code review item** rather than a manual test.

---

## X15. Batch Onboard — Mixed-School LRN Collision

> **Scenario:** LRN exists at School A, registrar tries to create same LRN at School B.

#### TC-X15.1: LRN already exists at School A — onboard at School B

Pre-condition: Student with LRN `120345678901` exists at School A.

```
POST {{base_url}}/students/onboard/
Authorization: Bearer {{token_b}}
Content-Type: multipart/form-data

school_id: {{school_b_id}}
first_name: Duplicate
last_name: Cross
lrn: 120345678901
```

**Expected:** `409 Conflict` — `DUPLICATE_LRN`. LRN uniqueness is global (not per-school), so this should fail.

**Verify:** The unique constraint `uq_student_lrn_alive` is not scoped to school — it's `(lrn)` WHERE `deleted_at IS NULL`.

---

#### TC-X15.2: Same LRN at same school (double-check)

```
lrn: 120345678901
school_id: {{school_a_id}}
```

**Expected:** `409 Conflict` — `DUPLICATE_LRN` (same as above, just verifying school doesn't matter for uniqueness).

---

## X16. Admin — Cross-School Object Save

> **Scenario:** A school-scoped admin user in Django Admin tries to save an object that belongs to another school.

#### TC-X16.1: Registrar A saves student at School B via admin

1. Login to `/admin/` as Registrar A (staff scoped to School A)
2. Navigate to Students → Add Student
3. Set `school` = School B
4. Fill required fields
5. Click Save

**Expected:** Error message: "Cannot save — object is outside your scope."

**This is already enforced** by `ScopedModelAdmin.save_model()`.

---

#### TC-X16.2: Registrar A views School B's students in admin

1. Login to `/admin/` as Registrar A
2. Navigate to Students list

**Expected:** Only School A's students visible. `SchoolHierarchyScopedQuerysetMixin` should scope the queryset.

---

## Severity Matrix

| TC-ID | Attack Vector | Expected | Severity | Status |
|---|---|---|---|---|
| TC-X1.1 | Section pref from wrong school | 400 | Medium | Likely PASS |
| TC-X3.1 | Program not offered by school | 400 | High | **NEEDS VERIFICATION** |
| TC-X4.1 | Admit with wrong school's section | 400 | **Critical** | **LIKELY FAILS** |
| TC-X4.2 | Admit with wrong term's section | 400 | High | **LIKELY FAILS** |
| TC-X6.1 | Admit with wrong program's section | 400 | High | **LIKELY FAILS** |
| TC-X7.1 | Onboard to wrong school (JSON) | 403 | **Critical** | **LIKELY FAILS** |
| TC-X7.2 | Onboard to wrong school (single) | 403 | **Critical** | **LIKELY FAILS** |
| TC-X7.3 | Onboard to wrong school (CSV) | 403 | **Critical** | **LIKELY FAILS** |
| TC-X8.1 | Rollover copies from wrong school | 400 | High | **LIKELY FAILS** |
| TC-X9.1 | Rollover targets wrong school | 403 | **Critical** | **LIKELY FAILS** |
| TC-X10.1 | Review wrong school's submission | 404 | Medium | Likely PASS |
| TC-X12.1 | Dropdown with wrong program | empty | Low | Likely PASS |
| TC-X13.1 | Mixed school section prefs | 400 | Medium | Likely PASS |
| TC-X15.1 | Cross-school LRN collision | 409 | Medium | Likely PASS |
| TC-X16.1 | Admin save outside scope | error | Medium | Likely PASS |

> **"LIKELY FAILS"** = code review shows no validation exists for this check. These should be run first and may require code fixes.

---

## Postman Automation Scripts

### Setup — Login as both registrars

Create a Postman folder called "Cross-School Setup" with these requests in order:

**Request 1: Login Registrar A**

```
POST {{base_url}}/auth/token/
```

```json
{ "username": "registrar_a", "password": "{{registrar_a_password}}" }
```

Post-response:

```javascript
if (pm.response.code === 200) {
    pm.environment.set('token_a', pm.response.json().access);
    console.log('Registrar A token saved');
}
```

**Request 2: Login Registrar B**

```json
{ "username": "registrar_b", "password": "{{registrar_b_password}}" }
```

Post-response:

```javascript
if (pm.response.code === 200) {
    pm.environment.set('token_b', pm.response.json().access);
    console.log('Registrar B token saved');
}
```

**Request 3: Get School A section offerings**

```
GET {{base_url}}/enrollment/section-offerings/?school={{school_a_id}}
Authorization: Bearer {{token_a}}
```

Post-response:

```javascript
const body = pm.response.json();
const results = body.results || body;
if (results.length > 0) {
    pm.environment.set('so_a_id', results[0].id);
    console.log('Saved so_a_id:', results[0].id);
}
```

**Request 4: Get School B section offerings**

```
GET {{base_url}}/enrollment/section-offerings/?school={{school_b_id}}
Authorization: Bearer {{token_b}}
```

Post-response:

```javascript
const body = pm.response.json();
const results = body.results || body;
if (results.length > 0) {
    pm.environment.set('so_b_id', results[0].id);
    console.log('Saved so_b_id:', results[0].id);
}
```

### Cross-school rejection assertion

Reuse on any cross-school test that should be rejected:

```javascript
pm.test('Cross-school request rejected', () => {
    pm.expect([400, 403, 404]).to.include(pm.response.code);
});

pm.test('No data corruption', () => {
    pm.expect(pm.response.code).to.not.eql(200);
    pm.expect(pm.response.code).to.not.eql(201);
});
```

### Cross-school admission rejection

```javascript
pm.test('Admission with wrong section rejected', () => {
    pm.expect(pm.response.code).to.be.oneOf([400, 403]);
    const body = pm.response.json();
    pm.expect(body.error).to.include('school');
});
```

### Folder structure

```
SMIS V2 — Cross-School Tests/
├── Setup/
│   ├── Login Registrar A
│   ├── Login Registrar B
│   ├── Get School A section offerings
│   └── Get School B section offerings
├── X1-X3. Enrollment Boundary/
│   ├── Cross-school section pref (X1.1)
│   ├── Program not offered (X3.1)
│   └── Mixed prefs (X13.1)
├── X4-X6. Admission Boundary/
│   ├── Wrong school's section (X4.1)
│   ├── Wrong term's section (X4.2)
│   └── Wrong program's section (X6.1)
├── X7. Onboarding Scope Bypass/
│   ├── JSON to wrong school (X7.1)
│   ├── Single to wrong school (X7.2)
│   ├── CSV to wrong school (X7.3)
│   └── Superuser cross-school (X7.4)
├── X8-X9. Rollover Boundary/
│   ├── Source term from other school (X8.1)
│   └── Target school outside scope (X9.1)
├── X10. Scoped Queryset/
│   ├── Review other school's submission (X10.1)
│   └── List other school's submissions (X10.2)
└── X15-X16. Edge Cases/
    ├── Cross-school LRN collision (X15.1)
    └── Admin scope bypass (X16.1)
```

---

## Known Gaps (Code Review Findings)

These are **not test failures** — they're architectural gaps identified during code review that the tests above are designed to surface.

| Gap | Location | Risk |
|---|---|---|
| `admit_enrollment_submission` doesn't validate `section_offering.section.school == submission.school` | `academics/services.py:233-238` | Student placed in wrong school's section |
| `admit_enrollment_submission` doesn't validate `section_offering.school_academic_term == submission.school_academic_term` | `academics/services.py:233-238` | Student placed in wrong year's section |
| `admit_enrollment_submission` doesn't validate `section_offering.academic_program == submission.academic_program` | `academics/services.py:233-238` | Grade 7 student in Grade 8 section |
| Student onboard views use `school_id` from request body, not user scope | `students/api/views.py` + serializers | Any authenticated user onboards to any school |
| `rollover_academic_year` doesn't validate `source_term.school == target school` | `academics/services.py:529-543` | Sections copied across schools |
| `RolloverApi` has no scope check | `academics/api/views.py:273-310` | Any user can rollover any school |
| `create_enrollment_submission` doesn't validate program is offered by school via `SchoolAcademicProgram` | `academics/services.py:94-96` | Submission for non-offered program |
| `SectionOffering.clean()` bypassed by `bulk_create` in rollover | `academics/models.py:177-221` | Cross-school section offerings via bulk |
| Bridge event `school_id` not cross-checked against resolved student's school | `realtime/handlers/campus_notification.py:44-51` | Notification via wrong school's config |

> **Recommendation:** Fix the critical gaps (onboard scope, admission cross-checks, rollover scope) before QA testing. The test cases above will then serve as regression tests.
