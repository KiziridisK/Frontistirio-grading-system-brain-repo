# Frontistirio — Grading System Brain

> **Full-stack reference** for the complete grading infrastructure: grade levels, categories, scales, scenarios, and the actual student course grade records. Covers backend models, routes, controller logic, and the Angular frontend.

---

## Change log

> Recorded 2026-09-14, verified against code. Older sections below describe the June/July design.

### 2026-09-21 — Test grading + performance report documented *(built 07-13 / 07-22, not recorded here until now)*
Verified against frontend `efc1260` / API `93b4079`. Two new sections at the end of this file:
- **[Test grading (`TestCourseGrade`)](#test-grading-testcoursegrade--built-2026-07-13)**: grading test-cycle tests,
  one score per (student, test, course), with the same approval flow.
- **[Student performance report («Επίδοση»)](#student-performance-report-επίδοση--built-2026-07-22)**: per-student
  PDF and preview for a month or date range, with trend (slope) + test grades.

Since the admin active-period override, the performance report resolves its period with `resolveActivePeriod`, not with `getStoreDefaultPeriod`.

### 2026-09-13 — Scale & scenario on the τάξη (`Grade`) itself
API `c5942d3`, frontend `384c942`.
- `models/grades.js` gained **`grade_scale: ObjectId → GradeScale`** and **`grade_scenario: ObjectId →
  GradeScenarios`** — the same collections a Course references. The old numeric `gradingScale` /
  `gradingScenario` fields (ids of a hardcoded client-side list, never read anywhere) are kept only so
  legacy documents load; they are marked legacy in both model files.
- `handlers/grades.js`: `createStoreGrade` casts the two ids (or leaves them `undefined`);
  `editStoreGrade` turns a cleared value (`''`/`null`) into `null` because `''` can't cast to an
  ObjectId.
- Frontend: `Grade` model `grade_scale?/grade_scenario?: string | null`; the redesigned
  `add-grade` form-modal (`.fm-*` shell) shows two single-choice chip groups ("Κλίμακα βαθμολόγησης" /
  "Σενάριο βαθμολόγησης") when scales/scenarios exist, with click-again-to-clear (`toggleChoice`).
- ⚠️ **Stored, not yet consumed**: nothing reads `Grade.grade_scale/grade_scenario` — courses still
  carry their own `grade_scale`/`grade_scenario`, the gradebook reads the course's, and a new course
  is not defaulted from its τάξη. Intended as the τάξη-level default; wiring is a follow-up.

### 2026-09-12 — Period coverage in labels + student grades page redesign + `/my-syllabus`
Frontend only (`384c942`).
- **`shared/period-timeline.util.ts` → `formatPeriodTimeline(key, translate, range?)`**: the optional
  `range: { from, to }` (a TeachingPeriod's `date_from/date_to`) makes non-monthly keys spell out the
  months they cover — `trimester-1` → **«Τριμηνιαία 1 (Ιούνιος 2026 – Αύγουστος 2026)»**. The months are
  reconstructed with the **same algorithm as the gradebook's `CourseGradeItemComponent.generatePeriods`**:
  `ceil(totalMonths / count)` months per slice, count = `trimester 3 / quarter 4 / semester 2`, last
  slice clamped to `to`; a one-month slice shows a single month. Without `range` the bare label is
  returned (back-compat). Monthly keys (`"2026-9"`) → «Σεπτέμβριος 2026» as before.
  `periodTimelineSortValue(key)` orders rows.
  - Callers passing the default period's span: `student/my-grades`, `home` (student latest grade),
    `parent/my-children`, `students/student-performance`, and `grade-approvals` (its private
    `formatPeriodKey` now delegates to the util; it subscribes to `selectTeachingPeriods`).
  - Limitation: a key like `trimester-1` stores **no dates**, so the months are always derived from
    the period's **current** `date_from/date_to` — editing a period's dates relabels old grades. The
    backend performance-report PDF does not use this labelling.
- **`/my-grades` redesign** (`student/my-grades/`): hero (period + courses / scores / overall average)
  → one card per course with average, trend arrow and sparkline → timeline of periods with delta chips
  (+1.5 / −0.5), «ΝΕΟ» (≤ 7 days) and «Τελευταία» tags, clamped comments; pull-to-refresh, retry on
  failure, cards for courses without grades yet. Shared chrome `student/_student-shell.scss`
  (`.student-page`, `.student-hero`, `.course-card`…) and `shared/course-accent.util.ts`
  `courseAccentClass(courseId)` → `accent-0…7` hashed from the id, so a course keeps one colour on
  every student page.
  - **Scores are deliberately not colour-toned** ("good/bad"): the student bootstrap branch doesn't ship
    `gradeScales`, so a bare 15 could be /20 or /100 — only the movement between periods is coloured.
- **`/my-syllabus`** (`student/my-syllabus/`, RoleGuard `['student']`): the ύλη tab moved out of
  `/my-grades` into its own page (menu entry + home quick-access card + cross-links between the two
  breadcrumb bars). The parent view (`my-children`) keeps its syllabus tab.

### 2026-07-07 — Student sees own grades (`/my-grades`) *(not previously recorded here)*
- `GET /student-course-grades/get-my-course-grades` (`isStudent`, `// NEW ROUTE`) →
  `getMyCourseGrades`: the student is resolved **from the login** (`getStudentByUserId(req.user.id,
  period)`), never from a client id; handler `getStudentOwnCourseGrades(storeId, studentId, periodId)` returns
  the period's records with **`status ≠ 'pending'`** (also matches legacy docs without the field) and
  `isDeleted:false`, grouped by `course_id`. The model's `visible` field is unused and not filtered.
- Frontend: `CourseGradeService.getMyCourseGrades()`, page `student/my-grades/` (course names from
  `selectAllCourses`, imports `RouterModule` for the breadcrumbs), home **latest-grade** card
  (`loadMyLatestGrade`, most recent by `updatedAt/createdAt`, score normalised to string so `0`
  renders), and the first student side-menu section (My grades / Educational material / Announcements).

---

## Grading System Layers

```
GradeCategory (e.g. "High School")        — superadmin creates
  └── Grade (e.g. "Grade 10")             — store uses, assigns to students/courses
        └── Course
              ├── grade_scale (e.g. 0–20) — how scores are expressed
              ├── grade_scenario (e.g. 30% midterm + 70% final) — how they combine
              └── StudentCourseGrade       — the actual recorded score per student
```

---

## Backend: MongoDB Models

### `models/grade-categories.js`
```javascript
{ name: String, store_id: ObjectId → Store }
```

### `models/grades.js`
```javascript
{
  name: String,
  store_id: ObjectId → Store,
  category: ObjectId → GradeCategory,
  description: String,
  grade_scale: ObjectId → GradeScale,          // 2026-09-13 (τάξη-level default, not consumed yet)
  grade_scenario: ObjectId → GradeScenarios,   // 2026-09-13
  gradingScale: Number, gradingScenario: Number, // legacy, unused
  isDeleted: Boolean, deletedAt: Date,
  createdBy/updatedBy: ObjectId → User
}
```

### `models/grade-scales.js`
```javascript
{
  name: String,       // e.g. "0–20 Numeric"
  min: Number,
  max: Number,
  store_id: ObjectId → Store
}
```

### `models/grade-scenarios.js`
```javascript
{
  name: String,
  description: String,   // e.g. "30% midterm + 70% final"
  store_id: ObjectId → Store
}
```

### `models/student-course-grade.js`
```javascript
{
  student_id: ObjectId → Student,
  course_id: ObjectId → Course,
  period_id: ObjectId → TeachingPeriod,
  store_id: ObjectId → Store,
  score: Mixed,           // the numeric/string score (NOT "grade" — see bugs GR-01)
  comment: Mixed,         // free-text performance comment
  visible: Boolean,       // student-visibility flag (still unused end-to-end)
  period_timeline: Mixed (required),  // the cell key, e.g. "2024-3" / "trimester-1"
  // --- Approval workflow (added 2026-07-06) ---
  status: "approved" | "pending" (default "approved"),
  submittedBy: ObjectId → User,   // teacher who proposed a pending change
  submittedAt: Date,
  createdBy/updatedBy: ObjectId → User,
  isDeleted: Boolean, deletedAt: Date,
  timestamps: true
}
```

---

## Backend: Routes

### `/grade-categories` (`routes/grade-categories.js`)
| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/grade-categories/get-grade-categories` | any authenticated | No role restriction — all roles can read |
| POST | `/grade-categories/create-grade-category` | superadmin | |
| POST | `/grade-categories/edit-grade-category` | superadmin | |
| DELETE | `/grade-categories/delete-grade-category` | superadmin | |

### `/grades` (`routes/grades.js`)
| Method | Path | Role |
|---|---|---|
| GET | `/grades/get-store-grades` | store-user |
| POST | `/grades/create-store-grade` | store-user |
| POST | `/grades/edit-store-grade` | store-user |
| DELETE | `/grades/delete-store-grade` | store-user |
| GET | `/grades/get-all` | **no auth** (open endpoint) |

### `/grade-scales` (`routes/grading-scales.js`)
| Method | Path | Role |
|---|---|---|
| GET | `/grade-scales/get-store-grade-scales` | store-user |
| POST | `/grade-scales/create-store-grade-scale` | store-user |
| POST | `/grade-scales/edit-store-grade-scale` | store-user |
| DELETE | `/grade-scales/delete-store-grade-scale` | store-user |

### `/grade-scenarios` (`routes/grade-scenarios.js`)
| Method | Path | Role |
|---|---|---|
| GET | `/grade-scenarios/get-store-grade-scenarios` | store-user |
| POST | `/grade-scenarios/create-store-grade-scenario` | store-user |
| POST | `/grade-scenarios/edit-store-grade-scenario` | store-user |
| DELETE | `/grade-scenarios/delete-store-grade-scenario` | store-user |

### `/student-course-grades` (`routes/student-course-grade.js`)
| Method | Path | Role |
|---|---|---|
| GET | `/student-course-grades/get-store-student-course-grades` | store-user |
| POST | `/student-course-grades/upsert-store-student-course-grades` | store-user |
| POST | `/student-course-grades/submit-student-course-grade` | **teacher** (approval workflow) |
| GET | `/student-course-grades/get-teacher-student-course-grades` | **teacher** — read a course's grades for the gradebook (added 2026-07-06) |
| GET | `/student-course-grades/get-pending-course-grades` | store-user (approval workflow) |
| GET | `/student-course-grades/get-pending-course-grades-count` | store-user (approval workflow) |
| POST | `/student-course-grades/approve-course-grade` | store-user (approval workflow) |
| POST | `/student-course-grades/reject-course-grade` | store-user (approval workflow) |

---

## Backend: Grade Controller Logic

### `getStoreStudentCourseGrades`
Requires query params: `course_id`, `student_ids[]`. Fetches grades for specific students in a course, scoped to the active default period. Returns results **grouped by `student_id`** using lodash `_.groupBy`:
```javascript
const grouped = _.groupBy(student_course_grades, 'student_id');
res.send({ success: true, student_course_grades: grouped });
```

### `createStoreStudentCourseGrades` (upsert)
Creates or updates a `StudentCourseGrade` record. Uses MongoDB `findOneAndUpdate` with `upsert: true` to handle both create and update in one operation.

### Grade Category Note
`getGradeCategories` has **no role restriction** (just `authMiddleware`) — any logged-in user can read categories. This is intentional so students/parents/teachers can also read grade level names. Creation/edit/delete require `superadmin`.

### Grade Handler Exports (`handlers/grades.js`)
```javascript
exports.createStoreGrade(gradeData, store_id, user_id)
exports.editStoreGrade(gradeData, store_id)
exports.fetchAllGrades()
exports.fetchStoreGrades(storeId)
exports.deleteStoreGrade(gradeId, permanently)
exports.watchGrades(io, userSockets)   // Change Stream
```

---

## Backend: `watchGrades`
- Emits `gradeAdded`, `gradeUpdated`, `gradeDeleted` to all store-users of the affected store
- Same pattern as other watchers

---

## Grade & Comment Approval Workflow (added 2026-07-06)

Lets teachers submit course grades/comments that an admin (store-user) must approve before they count,
gated by a store setting. Ahead of the (future) teacher view; the teacher-facing submit UI is deferred but
the backend submit endpoint is built and API-testable.

**Design (status-flag model, NOT a separate collection):** approval state lives on the existing
`StudentCourseGrade` via `status` + `submittedBy` + `submittedAt`. A teacher's pending submission
**overwrites the cell's current value** and marks it `pending`; the prior approved value is NOT preserved
(no history — accepted tradeoff). Admin approve → `approved`; reject → soft-delete.

**Store setting:** `StoreSettings.require_grade_approval` (Boolean, default false) — see the
`store-management` brain. When OFF, teacher submissions apply directly as `approved`.

**Backend flow:**
- `submitStudentCourseGrade` (controller, `isTeacher`): store_id from `req.user.store`; reads
  `require_grade_approval`; upserts the cell with `status:"pending"` + `submittedBy`/`submittedAt` when ON,
  else `approved`. Reuses the same find-or-create-or-update logic as the admin upsert;
  `createStoreStudentCourseGrades` gained a trailing `extra = { status, submittedBy, submittedAt }` arg and
  `updateStudentCourseGrade`'s whitelist gained `status/submittedBy/submittedAt/isDeleted/deletedAt`.
- `getPendingCourseGrades` / `getPendingCourseGradesCount` (`isStoreUser`): handler queries
  `{ store_id, period_id: activePeriod, status:"pending", isDeleted:false }` (new handler exports
  `getPendingStoreCourseGrades` + `countPendingStoreCourseGrades`).
- `approveCourseGrade` (`isStoreUser`): body `{ grade_id, score?, comment? }` → optional edit + `status:"approved"`, clears `submittedBy/submittedAt`.
- `rejectCourseGrade` (`isStoreUser`): soft-deletes the record (`isDeleted:true`).
- **Fixed bug GR-02 in passing:** `getStoreStudentCourseGrades` now filters `isDeleted:false`, so rejected
  (soft-deleted) grades disappear from the gradebook.

**Frontend (Frontistirio):**
- `CourseGrade` model gained `status`, `submittedBy`, `submittedAt`. `Teacher` model gained `user_id?`
  (to resolve `submittedBy` → teacher name). `StoreSettings` model gained `require_grade_approval?`.
- `CourseGradeService`: `submitStudentCourseGrade` (teacher, future UI), `getPendingCourseGrades`,
  `getPendingCourseGradesCount`, `approveCourseGrade(grade_id, score?, comment?)`, `rejectCourseGrade(grade_id)`.
- New standalone `GradeApprovalsComponent` at `/grade-approvals` (RoleGuard `['store-user']`): pending list,
  resolves student/course/teacher names + grade scale from NgRx, inline edit via the existing
  `GradePopoverComponent`/`CommentPopoverComponent`, Approve/Reject. period_timeline keys prettified
  client-side (`formatPeriodKey`).
- Settings toggle: `ion-toggle` bound to `store_settings.require_grade_approval` in
  `SetCourseSettingsComponent` (saves through the existing `upsertStoreSettings`).
- Home dashboard: `home.page.ts` fetches `getPendingCourseGradesCount()` once for store-user →
  red `ion-badge` on a new "Grade approvals" STUDIES card. **Simple count-on-load — no websocket/push.**
- i18n `menu.management.grade_approvals`, `course_settings.{grading_title,require_grade_approval,+hint}`,
  `course_grades.{no_pending,approve,reject,submitted_by}`, `messages.{grade_approved,grade_rejected,grade_action_failed}`.

### Teacher gradebook wired up (2026-07-06 — the "future teacher view" is now here)
The teacher-facing submit UI is **no longer deferred**: `CourseGradeItemComponent` (`/course-grades/:id`,
route roles += `teacher`) is now **role-aware** (reads `role` from `localStorage.user`):
- **Read:** teacher → `getTeacherStudentCourseGrades(student_ids, course_id)` (→ new
  `get-teacher-student-course-grades`), store-user → the existing `getStudentCourseGrades`. Same response shape.
- **Write:** a shared `persistGrade(studentId, periodKey, score, comment)` calls `submitStudentCourseGrade`
  (teacher; toasts `grade_submitted_pending` vs `grade_submitted_approved` from the returned `status`) or
  the direct `upsertStudentCourseGrade` (store-user).
- Teachers reach the gradebook from the new `/my-teaching` **Courses** tab (see teacher-management brain).
- **Ownership (server-side, added this session):** `submitStudentCourseGrade` AND the new read controller
  now validate `course_id ∈ teacher.period_courses[defaultPeriod]` via
  `teacherHandler.getTeacherByUserId` + `getTeacherPeriodAssignmentIds` (throws
  `course_not_assigned_to_teacher`). A shared `resolveTeacherForCourse(req, course_id)` helper in
  `controllers/student-course-grade.js` does the resolve+assert.
- i18n added: `messages.grade_submitted_{pending,approved}`.

**Deferred (still):** realtime/push for pending (a
`watchStudentCourseGrades` change-stream badge like `watchAnnouncements`); restoring the prior approved
value on reject.

---

## Frontend: TypeScript Models

```typescript
interface Grade {
  _id?: string; name: string; store_id?: string;
  category?: string | GradeCategory; isDeleted?: boolean;
}
interface GradeCategory { _id?: string; name: string; store_id?: string; }
interface GradeScale { _id?: string; name: string; min: number; max: number; store_id?: string; }
interface GradeScenario { _id?: string; name: string; description?: string; store_id?: string; }
interface CourseGrade {
  _id?: string; student_id: string; course_id: string;
  period_id?: string; store_id?: string; grade: number | string;
}
```

---

## Frontend: NgRx State

```typescript
AppState {
  grades: GradeState               // { grades: Grade[] }
  gradeCategories: GradeCategoryState  // { gradeCategories: GradeCategory[] }
  gradeScenarios: GradeScenarioState   // { gradeScenarios: GradeScenario[] }
  gradeScales: GradeScalesState    // { gradeScales: GradeScale[] }
}
```

All four are **populated at bootstrap** for store-users. GradeCategories and GradeScales/Scenarios are loaded from the backend globally (not store-scoped — shared configuration). Grades are store-specific.

**`StudentCourseGrade` records are NOT in NgRx** — fetched on-demand per course page.

Selectors: `selectAllGrades`, `selectGradeCategories`, `selectGradeScenarios`, `selectGradeScales`

---

## Frontend: Services

### `GradesService` (`services/grades.service.ts`)
```typescript
getStoreGrades()       → GET /grades/get-store-grades
addGrade(grade)        → POST /grades/create-store-grade
editGrade(grade)       → POST /grades/edit-store-grade
deleteGrade(id, perm)  → DELETE /grades/delete-store-grade  body: { id, permanently }
```

### `GradeCategoriesService`, `GradeScalesService`, `GradeScenariosService`
Same CRUD pattern for their respective endpoints.

### `CourseGradeService` (`services/course-grade-service.service.ts`)
```typescript
getCourseGrades(courseId, studentIds)
  → GET /student-course-grades/get-store-student-course-grades
    ?course_id=xxx&student_ids[]=yyy&student_ids[]=zzz

upsertCourseGrade(grade)
  → POST /student-course-grades/upsert-store-student-course-grades
  body: { student_id, course_id, period_key, score, comment }   // NOT "grade"/"period_id" — see bugs GR-01

// Approval workflow (2026-07-06):
submitStudentCourseGrade(student_id, course_id, period_key, score?, comment?)
  → POST /student-course-grades/submit-student-course-grade      // teacher; pending vs approved by store setting
getPendingCourseGrades()          → GET  /student-course-grades/get-pending-course-grades
getPendingCourseGradesCount()     → GET  /student-course-grades/get-pending-course-grades-count
approveCourseGrade(grade_id, score?, comment?)  → POST /student-course-grades/approve-course-grade
rejectCourseGrade(grade_id)       → POST /student-course-grades/reject-course-grade
```

---

## Frontend: Components

### `GradesComponent` (`/grades`) + `AddGradeComponent` + `GradeItemComponent`
Basic list/create/edit for grade levels. Accessible to any authenticated user.

### `GradeCategoriesComponent` (`/grade-categories`) — superadmin only
List/create/edit grade categories. Route guarded by `RoleGuard` with `['superadmin']`.

### `GradeScalesComponent` (`/grade-scales`) + `AddGradeScaleComponent` + `GradeScaleItemComponent`
Form for min/max + name. Superadmin only.

### `GradeScenariosComponent` (`/grade-scenarios`) + `AddGradeScenarioComponent`
Form for name + description. Superadmin only.

### `CourseGradesComponent` (`/course-grades`)
Lists all courses. Click a course → `CourseGradeItemComponent`.

### `CourseGradeItemComponent` (`/course-grades/:id`) — 12KB TS + 1.3KB HTML
- **Now role-aware** (store-user + teacher) — see "Teacher gradebook wired up" above.
- Receives `courseId` from route
- Fetches all students enrolled in the course for active period (from NgRx)
- Calls `getCourseGrades(courseId, studentIds)` → grouped grade records
- Renders editable grade fields per student
- On change → `upsertCourseGrade()` — creates or updates the record

### `GradePopoverComponent`
Lightweight popover used in student/course detail views to display a grade value inline.

### `CommentPopoverComponent`
Popover for adding comments to individual grade records.

---

## Grade Display in Other Components

Student's overall grade level per period:
```typescript
// Read from student.period_grade for active period:
const gradeEntry = student.period_grade?.find(pg => pg.period === defaultPeriod._id);
const gradeId = gradeEntry?.grade; // ObjectId string
const grade = grades.find(g => g._id === gradeId); // from NgRx selectAllGrades
```

Course-level scores (StudentCourseGrade) are separate and only shown in `CourseGradeItemComponent`.

---

## Test grading (`TestCourseGrade`) — built 2026-07-13

> Verified 2026-09-21 @ frontend `efc1260` / API `93b4079`.

Grades **test-cycle tests** in parallel with course grading. A test is one exam on one day, so the gradeable unit is
**one score + one comment per (student, test, course)**. There are **no scenario or period columns** (the user's choice).
A `Test` can have several `course_ids`, so the unit is really (test × course). Participants are the students whose
active-period courses include that course.

**Approval:** same as course grades. It uses the same store setting **`require_grade_approval`**: a teacher's submission is saved as
`pending` or `approved`, and an admin write is always `approved`. **Reject = soft-delete the pending cell.**

### Model — `models/test-course-grade.js` (collection `TestCourseGrade`)
`store_id`, `period_id`, `test_id`, `course_id`, `student_id`, `test_cycle_id` (denormalized), `score`/`comment` (Mixed),
`status` (`approved` default | `pending`), `submittedBy`/`submittedAt`, `isDeleted`/`deletedAt`, timestamps.
Indexes: **partial-unique** `{store_id, period_id, test_id, course_id, student_id}` on `isDeleted:false` (the cell)
and `{store_id, period_id, status}` (approval list + badge).

### Routes — `/test-course-grades` (`routes/test-course-grade.js`, all `// NEW ROUTE`)
| Path | Guard | Controller |
|---|---|---|
| GET `/get-store-test-course-grades` | isStoreUser | `getStoreTestCourseGrades` |
| GET `/get-teacher-test-course-grades` | isTeacher | `getTeacherTestCourseGrades` |
| GET `/get-store-gradeable-cycles` | isStoreUser | `getStoreGradeableCycles`: cycle → tests listing |
| GET `/get-teacher-gradeable-cycles` | isTeacher | `getTeacherGradeableCycles`: only the teacher's courses; no date filter |
| POST `/upsert-store-test-course-grade` | isStoreUser | `upsertStoreTestCourseGrade`: always `approved` |
| POST `/submit-test-course-grade` | isTeacher | `submitTestCourseGrade`: `pending`/`approved` from the setting |
| GET `/get-pending-test-course-grades` (+ `-count`) | isStoreUser | approval list + badge count |
| POST `/approve-test-course-grade`, `/reject-test-course-grade` | isStoreUser | approve / soft-delete |

Controller helpers: `loadTestOrThrow` checks that `course_id ∈ test.course_ids`. `resolveTeacher`/`resolveTeacherForTestCourse`
check that the teacher owns the course this period **and** that the course is in the test.
**Teacher bootstrap does not include `test_cycles`**, which is why the teacher has a separate gradeable-cycles endpoint.

### Frontend
- `models/test-course-grade.model.ts` (`TestCourseGrade`, `GradeableCycle`, `GradeableTest`) and `services/test-course-grade.service.ts`
  (reads and writes per role, gradeable cycles, pending/count/approve/reject).
- **`test-grades/test-grades-list/`** (`@Input() mode: 'store-user' | 'teacher'`): cycle → test day → one tile per course, each linking to the gradebook.
  It is used in two places:
  - `/course-grades` (admin) has an `ion-segment` «Μαθήματα | Διαγωνισμοί».
  - `/my-teaching` (teacher) has a «Διαγωνισμοί» tab (`TESTS`).
- **`test-grades/test-grade-item/`** → route **`/test-grades/:testId/:courseId`** (store-user, teacher). One row per student with **one**
  grade cell (`GradePopover`, min/max from `course.grade_scale`) + a comment (`CommentPopover`). A teacher **submits**
  (toast says pending or approved); an admin **upserts**. `goBack()` = `history.back`, because the list lives on two pages.
- **`/grade-approvals`** has a second section «Διαγωνισμοί» for pending `TestCourseGrade`s. The home badge
  (`loadPendingApprovalsCount`) is the sum of course and test pending counts.
- i18n: `course_grades.{tab_courses,tab_tests}`, `test_grades.*`, `teacher.tabs.tests`.

### Gaps
- **Students and parents do not see their test grades** anywhere except inside the admin's performance report below.
- No realtime. Not verified end to end (grading a test, pending → approve/reject, the badge).

---

## Student performance report («Επίδοση») — built 2026-07-22

> Verified 2026-09-21 @ frontend `efc1260` / API `93b4079`.

The admin (store-user only) generates a **report for one student** in the active period, for **one month** or a **date range**.
It gives a live preview plus a **PDF**, across **all** the student's courses. For each course it shows the **linear-regression trend (slope)**
of the grades in the range, and it lists the **διαγωνίσματα** (test grades) whose test date falls in the range.

**Key design:** the backend builds **one structure** for both the JSON preview and the PDF, so they cannot drift.
The same pattern is used by the parent-meetings and absence reports.

### How the builder works — `helpers/buildStudentPerformanceReport.js`
`buildStudentPerformanceReport({storeId, studentId, period, from, to})`:
1. Courses = **`getStudentIdPeriodCourseIds`** (the student's own courses ∪ their τμήμα's; see `Frontistirio-course-syllabus-brain-repo` §5).
2. Grades are keyed by `period_timeline` (`"2024-9"` for monthly, `"trimester-1"` for ranged, depending on the course's `grade_scenario`).
   **`resolveTimelineKey`** turns each key into a concrete `[start, end]` interval. It mirrors the frontend
   `CourseGradeItemComponent.generatePeriods` (`SCENARIO_COUNT` trimester 3 / quarter 4 / semester 2). A cell is kept if its interval
   **overlaps** the selected range.
3. Only grades with `status ≠ pending` and `isDeleted: false` are used, for both `StudentCourseGrade` and `TestCourseGrade`. This is the same
   filter as the student and parent views.
4. **`computeTrend`**: least-squares slope + delta (last − first) + direction up/down/flat (`EPS = 0.05`). It returns `null` when there are fewer
   than 2 numeric points. A single month usually has one cell per course, so **no trend**. A 3-month range usually shows one.
5. Tests: `TestCourseGrade` → `Test.date` (+ `TestCycle.description`), filtered by date-in-range, grouped per course.
Also exports `resolveTimelineKey`, `computeTrend`, `GREEK_MONTHS`.

### PDF — `helpers/createPerformanceReportPdf.js`
PDFKit + DejaVuSans for Greek. One section per course: trend line (↑/↓/→ + delta + slope), average, grades table
(Περίοδος | Βαθμός | Σχόλιο), tests table (Ημ/νία | Διαγώνισμα | Βαθμός | Σχόλιο). Page overflow is handled with `heightOfString`.

### Routes — `/performance-reports` (`routes/performance-report.js`, `// NEW ROUTE`, both `isStoreUser`)
| Path | Controller |
|---|---|
| GET `/get-student-performance-report?student_id&from&to` | `getStudentPerformanceReport`: JSON for the preview |
| POST `/export-student-performance-report` | `exportStudentPerformanceReport`: PDF → S3 `logeion-test-cycle-reports`, prefix `performance-reports/<group>/<store>/`, signed URL valid 60 s |

The period comes from `resolveActivePeriod`. **`parseLocalDate`** parses `"YYYY-MM-DD"` as the **local** start or end of the day
(not `new Date()`, which is UTC), so the range bounds line up with the locally built cell intervals. Without `from`/`to`, the range is the whole period.

### Frontend
- **`students/student-performance/`** (standalone; `@Input() student`, `@Input() period`) is loaded in `student-details` under the
  **`PERFORMANCE`** tab. It is imported in **`app.module.ts`** because `StudentDetailsComponent` is declared in the module, not standalone.
  It has a Μήνας | Χρονική περίοδος `ion-segment`, `ion-datetime-button` + `ion-modal` pickers limited to the period's `date_from`/`date_to`,
  a live preview that reloads on every change, and an Export PDF button (`triggerDownload`). Preview labels use `formatPeriodTimeline`.
- `models/performance-report.model.ts`, `services/performance-report.service.ts`. i18n: top-level `performance.*`.

### Gaps
- No batch export for all students, and no range across periods (the report covers only the active period).
- Admin only: students and parents cannot see it.
- Not verified end to end (preview, trend arrows, PDF download).
