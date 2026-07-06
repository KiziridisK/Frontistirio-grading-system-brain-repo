# Frontistirio — Grading System Brain

> **Full-stack reference** for the complete grading infrastructure: grade levels, categories, scales, scenarios, and the actual student course grade records. Covers backend models, routes, controller logic, and the Angular frontend.

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

**Deferred (with the teacher view):** teacher submit screen; realtime/push for pending (a
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
