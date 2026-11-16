
# 01 — Database Normalization: LearnHub

**Project:** LearnHub_AP_2026  
**Database model:** Relational database  
**Proposed DBMS:** PostgreSQL  
**Scope:** Users, Courses, Lessons, Enrollments, Payments, Reviews, and Learning Progress

---

## 1. Objective

The objective of normalization is to organize LearnHub's data into related tables so that:

- the same information is not unnecessarily repeated;
- insert, update, and delete anomalies are reduced;
- each fact is stored in an appropriate place;
- relationships are enforced using primary keys and foreign keys;
- data remains consistent as students enroll, pay, review courses, and track lesson completion.

The repository description identifies the seven modules listed above. It does not prescribe every column or business rule. Therefore, the attributes and rules in this document are a proposed database design for the project and should be confirmed with the team before implementation.

## 2. Main entities and attributes

### 2.1 Users

Stores all account types in one table. The `role` attribute distinguishes students, teachers, and administrators.

| Attribute | Description | Constraint |
|---|---|---|
| user_id | Unique user identifier | Primary key |
| full_name | User's name | NOT NULL |
| email | Login email | UNIQUE, NOT NULL |
| password_hash | Hashed password | NOT NULL |
| role | student, teacher, or admin | CHECK |
| bio | Optional profile description | Optional |
| is_active | Whether account is active | Default TRUE |
| created_at | Account creation time | Default current timestamp |

### 2.2 Courses

| Attribute | Description | Constraint |
|---|---|---|
| course_id | Unique course identifier | Primary key |
| teacher_id | Teacher who owns the course | FK → Users |
| title | Course title | NOT NULL |
| description | Course description | Optional |
| price | Course price | Non-negative |
| course_status | draft, published, or archived | CHECK |
| created_at | Creation time | Default current timestamp |

### 2.3 Lessons

| Attribute | Description | Constraint |
|---|---|---|
| lesson_id | Unique lesson identifier | Primary key |
| course_id | Parent course | FK → Courses |
| title | Lesson title | NOT NULL |
| description | Lesson description | Optional |
| content_url | Video/content location | Optional |
| duration_seconds | Lesson duration | Non-negative |
| lesson_order | Position in the course | Unique within a course |

### 2.4 Enrollments

Resolves the many-to-many relationship between students and courses.

| Attribute | Description | Constraint |
|---|---|---|
| enrollment_id | Unique enrollment identifier | Primary key |
| student_id | Enrolled student | FK → Users |
| course_id | Selected course | FK → Courses |
| enrolled_at | Enrollment date/time | Default current timestamp |
| enrollment_status | active, completed, or cancelled | CHECK |

A student can enroll in a particular course only once. This is enforced by `UNIQUE(student_id, course_id)`.

### 2.5 Payments

Stores payment transactions or attempts related to an enrollment.

| Attribute | Description | Constraint |
|---|---|---|
| payment_id | Unique payment identifier | Primary key |
| enrollment_id | Related enrollment | FK → Enrollments |
| amount | Amount paid/attempted | Non-negative |
| currency | Currency code | Default INR |
| payment_method | UPI, card, net banking, wallet, other | CHECK |
| transaction_reference | Gateway reference | UNIQUE when present |
| payment_status | pending, successful, failed, refunded | CHECK |
| paid_at | Successful payment time | Optional |

One enrollment may have multiple payment attempts, for example when an initial payment fails and the student retries.

### 2.6 Reviews

Stores a student's feedback for an enrolled course.

| Attribute | Description | Constraint |
|---|---|---|
| review_id | Unique review identifier | Primary key |
| enrollment_id | Enrollment being reviewed | FK → Enrollments, UNIQUE |
| rating | Rating from 1 to 5 | CHECK |
| review_text | Written feedback | Optional |
| created_at | Review creation time | Default current timestamp |

The unique enrollment reference permits at most one review per enrollment.

### 2.7 Learning_Progress

Stores progress for each lesson in a student's enrolled course.

| Attribute | Description | Constraint |
|---|---|---|
| progress_id | Unique progress identifier | Primary key |
| enrollment_id | Student's enrollment | FK → Enrollments |
| course_id | Course of the enrollment | Used in composite FK |
| lesson_id | Lesson being tracked | Used in composite FK |
| progress_status | not_started, in_progress, completed | CHECK |
| watched_seconds | Time watched | Non-negative |
| completed_at | Completion time | Optional |
| updated_at | Last update time | Default current timestamp |

A student can have one progress record per lesson in an enrollment. The pair `(enrollment_id, lesson_id)` is unique.

---

## 3. Functional dependencies

A functional dependency `X → Y` means that a value of X determines exactly one value of Y.

The main dependencies in the proposed design are:

| Relation | Functional dependency |
|---|---|
| Users | `user_id → full_name, email, password_hash, role, bio, is_active, created_at` |
| Users | `email → user_id, full_name, password_hash, role, bio, is_active, created_at` |
| Courses | `course_id → teacher_id, title, description, price, course_status, created_at` |
| Lessons | `lesson_id → course_id, title, description, content_url, duration_seconds, lesson_order` |
| Lessons | `(course_id, lesson_order) → lesson_id, title, description, content_url, duration_seconds` |
| Enrollments | `enrollment_id → student_id, course_id, enrolled_at, enrollment_status` |
| Enrollments | `(student_id, course_id) → enrollment_id, enrolled_at, enrollment_status` |
| Payments | `payment_id → enrollment_id, amount, currency, payment_method, transaction_reference, payment_status, paid_at` |
| Reviews | `review_id → enrollment_id, rating, review_text, created_at` |
| Reviews | `enrollment_id → review_id, rating, review_text, created_at` |
| Learning_Progress | `progress_id → enrollment_id, course_id, lesson_id, progress_status, watched_seconds, completed_at, updated_at` |
| Learning_Progress | `(enrollment_id, lesson_id) → progress_id, course_id, progress_status, watched_seconds, completed_at, updated_at` |

These dependencies follow from the primary keys and unique constraints proposed for the schema.

---

## 4. UNF — Unnormalized Form

Imagine storing all LearnHub information in one large record called `LEARNHUB_RECORD`.

A simplified example could look like this:

| student_id | student_name | course_id | course_title | teacher_name | lessons | payments | review | progress |
|---|---|---|---|---|---|---|---|---|
| 101 | Asha | 501 | Web Development | Ravi | [(1, HTML), (2, CSS)] | [(P1, 499, success)] | (5, Great course) | [(1, completed), (2, in progress)] |

**Problems in UNF:**

1. `lessons` contains multiple lesson values in one field.
2. `payments` contains multiple payment values in one field.
3. `progress` contains multiple progress values in one field.
4. Student, course, and teacher information is repeated when a student has multiple courses.
5. Updating a course title in many records can create inconsistent data.

UNF is not a suitable relational design because it contains repeating groups and multi-valued attributes.

---

## 5. 1NF — First Normal Form

A relation is in 1NF when:

1. each column contains a single atomic value;
2. there are no lists or repeating groups inside a cell;
3. each row can be uniquely identified.

To reach 1NF, lesson, payment, and progress values must be represented as individual rows rather than arrays or lists.

For example, lesson data becomes:

| course_id | lesson_id | lesson_title | lesson_order |
|---|---|---|---|
| 501 | 1 | HTML Basics | 1 |
| 501 | 2 | CSS Basics | 2 |

Payment data becomes:

| payment_id | enrollment_id | amount | payment_status |
|---|---|---:|---|
| 9001 | 7001 | 499.00 | successful |
| 9002 | 7002 | 699.00 | pending |

Progress data becomes:

| enrollment_id | lesson_id | progress_status | watched_seconds |
|---|---|---|---:|
| 7001 | 1 | completed | 600 |
| 7001 | 2 | in_progress | 240 |

**Result:** Every field is atomic and repeating groups are represented as separate rows. However, keeping all attributes together in a large flattened relation would still cause duplication. We decompose the data further.

---

## 6. 2NF — Second Normal Form

A relation is in 2NF when:

1. it is already in 1NF;
2. every non-key attribute depends on the whole candidate key, not only part of a composite candidate key.

Consider a flattened enrollment/lesson relation with candidate key:

`(student_id, course_id, lesson_id)`

Some dependencies are:

- `student_id → student_name, student_email`
- `course_id → course_title, teacher_id`
- `lesson_id → lesson_title, lesson_duration` (assuming globally unique lesson IDs)
- `(student_id, course_id) → enrolled_at`
- `(student_id, course_id, lesson_id) → progress_status, watched_seconds`

Student details depend only on `student_id`, and course details depend only on `course_id`. These are partial dependencies on parts of the composite key.

**Decomposition to remove partial dependencies:**

### USERS

`USERS(user_id, full_name, email, password_hash, role, bio, is_active, created_at)`

Primary key: `user_id`

### COURSES

`COURSES(course_id, teacher_id, title, description, price, course_status, created_at)`

Primary key: `course_id`

### LESSONS

`LESSONS(lesson_id, course_id, title, description, content_url, duration_seconds, lesson_order)`

Primary key: `lesson_id`

### ENROLLMENTS

`ENROLLMENTS(enrollment_id, student_id, course_id, enrolled_at, enrollment_status)`

Primary key: `enrollment_id`  
Additional unique key: `(student_id, course_id)`

### LEARNING_PROGRESS

`LEARNING_PROGRESS(progress_id, enrollment_id, course_id, lesson_id, progress_status, watched_seconds, completed_at, updated_at)`

Primary key: `progress_id`  
Additional unique key: `(enrollment_id, lesson_id)`

Payments and reviews are also kept in their own relations because their attributes describe payment transactions and reviews, not the student, course, or lesson itself.

**Result:** Attributes are stored with the entity or relationship they describe. Partial dependencies on the composite enrollment/lesson key are removed.

---

## 7. 3NF — Third Normal Form

A relation is in 3NF when:

1. it is already in 2NF;
2. no non-key attribute depends transitively on a candidate key through another non-key attribute.

For example, if a flattened course record stores:

`course_id, teacher_id, teacher_name, teacher_email`

and the dependencies are:

- `course_id → teacher_id`
- `teacher_id → teacher_name, teacher_email`

then:

`course_id → teacher_id → teacher_name, teacher_email`

This is a transitive dependency. Teacher details should be stored in `USERS`, while `COURSES` stores only `teacher_id` as a foreign key.

Similarly:

- student name and email are stored in `USERS`, not repeated in `ENROLLMENTS`;
- course title and price are stored in `COURSES`, not repeated in `ENROLLMENTS` or `PAYMENTS`;
- lesson title and duration are stored in `LESSONS`, not repeated in progress records;
- payment details are stored in `PAYMENTS`;
- review details are stored in `REVIEWS`.

**Result:** The proposed relations are in 3NF under the stated functional dependencies. Non-key attributes depend on the key, the whole key, and nothing but the key.

---

## 8. BCNF — Boyce-Codd Normal Form

A relation is in BCNF when, for every non-trivial functional dependency `X → Y`, X is a superkey.

Check the proposed relations:

| Relation | Determinant / candidate key | BCNF check |
|---|---|---|
| USERS | `user_id`; also unique `email` | Every determinant is a candidate key |
| COURSES | `course_id` | Primary key determines course attributes |
| LESSONS | `lesson_id`; also `(course_id, lesson_order)` | Both determinants are candidate keys |
| ENROLLMENTS | `enrollment_id`; also `(student_id, course_id)` | Both determinants are candidate keys |
| PAYMENTS | `payment_id` | Primary key determines payment attributes |
| REVIEWS | `review_id`; also `enrollment_id` | Both determinants are candidate keys |
| LEARNING_PROGRESS | `progress_id`; also `(enrollment_id, lesson_id)` | Both determinants are candidate keys |

**Result:** The proposed schema satisfies BCNF for the functional dependencies and uniqueness rules listed in this document. If new business rules introduce additional dependencies, the BCNF analysis must be revisited.

---

## 9. Final normalized relational schema

The final design contains seven relations:

1. `USERS(user_id PK, full_name, email UNIQUE, password_hash, role, bio, is_active, created_at, updated_at)`
2. `COURSES(course_id PK, teacher_id FK, title, description, price, course_status, created_at, updated_at)`
3. `LESSONS(lesson_id PK, course_id FK, title, description, content_url, duration_seconds, lesson_order, UNIQUE(course_id, lesson_order))`
4. `ENROLLMENTS(enrollment_id PK, student_id FK, course_id FK, enrolled_at, enrollment_status, UNIQUE(student_id, course_id))`
5. `PAYMENTS(payment_id PK, enrollment_id FK, amount, currency, payment_method, transaction_reference UNIQUE, payment_status, paid_at, created_at)`
6. `REVIEWS(review_id PK, enrollment_id FK UNIQUE, rating, review_text, created_at, updated_at)`
7. `LEARNING_PROGRESS(progress_id PK, enrollment_id FK, course_id, lesson_id, progress_status, watched_seconds, completed_at, updated_at, UNIQUE(enrollment_id, lesson_id))`

### Relationship summary

- One teacher can teach many courses.
- One course can contain many lessons.
- Students and courses have a many-to-many relationship through enrollments.
- One enrollment can have multiple payment attempts.
- One enrollment can have zero or one review.
- One enrollment can have multiple lesson-progress records.

---

## 10. Data integrity constraints

The final design should enforce these rules:

1. `users.email` must be unique.
2. `users.role` must be one of `student`, `teacher`, or `admin`.
3. A course must reference an existing user as its teacher. The application must additionally verify that the referenced user has the `teacher` role.
4. A lesson must reference an existing course.
5. A student-course pair must be unique in `enrollments`.
6. A lesson's order must be unique within its course.
7. A payment must reference an existing enrollment.
8. A review rating must be between 1 and 5.
9. An enrollment can have at most one review.
10. A progress record must refer to a lesson belonging to the same course as the enrollment.
11. Watched duration and course price cannot be negative.

---

## 11. Conclusion

The LearnHub database has been organized into seven related tables: `USERS`, `COURSES`, `LESSONS`, `ENROLLMENTS`, `PAYMENTS`, `REVIEWS`, and `LEARNING_PROGRESS`.

The design proceeds from UNF to 1NF by removing repeating groups and ensuring atomic values; to 2NF by removing partial dependencies; to 3NF by removing transitive dependencies; and to BCNF by ensuring every determinant in the stated functional dependencies is a candidate key or superkey.

The result reduces unnecessary duplication and supports referential integrity for the core LearnHub learning workflow.
