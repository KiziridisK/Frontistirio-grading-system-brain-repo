# Frontistirio — Grading System Brain

> **Purpose:** Documents the full grading system including grades (school year levels), grade categories, grade scales, grade scenarios, and student course grades. Reference this when working on anything related to evaluation or grade configuration.

---

## Overview

The grading system has multiple layers:

```
Grade (school level, e.g. "1st High School")
  └── GradeCategory (grouping of grades, e.g. "High School")
        └── GradeScenario (how scores combine → final grade)
              └── GradeScale (the scale used, e.g. 0-20)
                    └── StudentCourseGrade (actual student score per course)
```

---

## Models

### Grade (`models/grades.js`)
School grade/year level (e.g., "Grade 10", "1st Gymnasium"):
```
Grade {
  name       String
  store_id   → Store
  category   → GradeCategory
  isDeleted  Boolean
}
```

### GradeCategory (`models/grade-categories.js`)
Groups grades together (e.g., "Elementary", "High School"):
```
GradeCategory {
  name      String
  store_id  → Store
}
```

### GradeScale (`models/grade-scales.js`)
The scoring scale for a course:
```
GradeScale {
  name      String      // e.g., "0-20 Numeric"
  min       Number
  max       Number
  store_id  → Store
}
```

### GradeScenario (`models/grade-scenarios.js`)
Defines how multiple test scores combine into a final grade:
```
GradeScenario {
  name          String
  description   String
  store_id      → Store
  // (weights/rules for combining scores)
}
```

### StudentCourseGrade (`models/student-course-grade.js`)
Actual grade record — one per student per course per period:
```
StudentCourseGrade {
  student_id  → Student
  course_id   → Course
  period_id   → TeachingPeriod
  store_id    → Store
  grade       Number (or String depending on scale)
  createdBy / updatedBy → User
}
```

---

## API Endpoints

### Grades (`routes/grades.js`)

| Method | Path | Description |
|---|---|---|
| POST | `/grades/` | Create a grade level |
| PUT | `/grades/:id` | Edit a grade level |
| DELETE | `/grades/:id` | Delete a grade level |
| GET | `/grades/` | Get all store grades |

### Grade Categories (`routes/grade-categories.js`)

| Method | Path | Description |
|---|---|---|
| POST | `/grade-categories/` | Create category |
| PUT | `/grade-categories/:id` | Edit category |
| DELETE | `/grade-categories/:id` | Delete category |
| GET | `/grade-categories/` | Get all categories |

### Grade Scales (`routes/grading-scales.js`)

| Method | Path | Description |
|---|---|---|
| POST | `/grade-scales/` | Create scale |
| PUT | `/grade-scales/:id` | Edit scale |
| GET | `/grade-scales/` | Get all scales |

### Grade Scenarios (`routes/grade-scenarios.js`)

| Method | Path | Description |
|---|---|---|
| POST | `/grade-scenarios/` | Create scenario |
| PUT | `/grade-scenarios/:id` | Edit scenario |
| GET | `/grade-scenarios/` | Get all scenarios |

### Student Course Grades (`routes/student-course-grade.js`)

| Method | Path | Description |
|---|---|---|
| POST | `/student-course-grades/` | Record a grade |
| PUT | `/student-course-grades/:id` | Update a grade |
| GET | `/student-course-grades/student/:studentId` | Get student's grades |
| GET | `/student-course-grades/course/:courseId` | Get all grades for a course |

---

## How Grading Works End-to-End

### Setup (done by store admin)
1. Create **GradeCategories** (e.g., "Gymnasium", "Lyceum")
2. Create **Grades** within categories (e.g., "A' Gymnasium")
3. Create **GradeScales** (e.g., 0–20 numeric)
4. Create **GradeScenarios** (e.g., "30% midterm + 70% final")
5. Assign `grade_id`, `grade_scale`, `grade_scenario` to each **Course**

### Recording Grades
1. Store-user (or teacher) records a `StudentCourseGrade`
2. Linked to `student_id`, `course_id`, `period_id`
3. Test cycle scores (from `TestCycle`/`Test`) feed into scenario calculation

### Student Grade per Period
- `Student.period_grade` stores their overall **school grade level** (not test score) per period
- Per-course scores are in `StudentCourseGrade`

---

## Grade Scenario Logic

The scenario defines weighted combination of test scores:
- Multiple `Test` documents belong to a `TestCycle`
- `TestCycle` is scoped to `grade_ids` and `course_ids`
- `StudentCourseGrade` represents the computed/recorded final result

---

## Test Cycles & Tests (brief — see `test-cycles` brain for full detail)

Test cycles group individual tests:
```
TestCycle → [Test, Test, Test] → StudentCourseGrade (calculated)
```
