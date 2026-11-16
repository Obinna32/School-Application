## System Architecture & Structural Entities

The platform separates user identity from operational context. Every person logging into the portal has a single system identity that handles credentials and security, but their actual interactions within the system are governed by distinct domain profiles.

### Core System Entities & Their Relationships

1. **User Identity**
The core account layer. Every individual—whether a Super Admin, Teacher, Student, or Parent—possesses a single identity containing login credentials, account status, and assigned platform roles.
2. **Academic Year & Terms**
The temporal boundary of the school. An Academic Year (such as 2026–2027) contains multiple Terms or Semesters. Terms hold operational flags, such as whether grade entry is open or locked.
3. **Grade Levels & Class Sections**
The organizational grouping of students. A Grade Level (e.g., Grade 10) contains one or more Class Sections (e.g., 10-A, 10-B). Each Class Section is assigned a Form Teacher (or Homeroom Teacher) who acts as the primary administrative overseer for that specific student group.
4. **Subjects**
The courses taught within the school (e.g., Mathematics, Biology, History).
5. **Class-Subject Allocations**
The operational bridge linking a Class Section, a Subject, and an assigned Subject Teacher. For instance, "Mr. Smith (Teacher) teaches Mathematics (Subject) to Grade 10-A (Class Section)." This entity is critical for authorization: it defines exactly which teacher has authority to record grades and attendance for a specific group of students.
6. **Student Profiles**
Linked one-to-one with a User Identity. A Student Profile holds enrollment details, admission numbers, and a direct association with exactly one active Class Section per academic year.
7. **Parent Profiles**
Linked one-to-one with a User Identity. A Parent Profile holds contact records and has a many-to-many relationship with Student Profiles through explicit mapping. A single parent account can be mapped to multiple students (siblings), and a single student can be linked to multiple parents or legal guardians.
8. **Attendance Records**
Daily or subject-level attendance logs linked to a Student, a Class Section, a specific date, and the User Identity of the person who took attendance.
9. **Assessment & Grade Records**
Individual evaluation records linked to a Student, a Class-Subject Allocation, an Academic Term, an Assessment Category (e.g., Homework, Midterm, Final Exam), and the recording Teacher.
10. **Billing & Fee Records**
Financial items, invoices, and payment tracking linked directly to a Student Profile, accessible by parents and finance administrators.
11. **System Audit Logs**
Immutable historical records capturing every sensitive action across the platform—notably grade overrides, privilege changes, and term locking events.

---

## Detailed Roles & Permission Architecture

The authorization model relies on **Resource-Action-Scope** permissions. Access is granted not just based on *what action* a user wants to perform, but *within what scope* they are permitted to perform it.

### 1. Super Admin (System Owner / IT Administrator)

* **Scope:** System-wide across all school settings and database tables.
* **Responsibilities:** System configuration, global user management, platform audit review, role assignments, security policies.
* **Permissions:**
* User Accounts: Full create, read, update, delete, suspend, and password reset access across all accounts.
* System Configuration: Access to global parameters, term dates, backup schedules, and role definitions.
* Audit Logs: Read-only access to all system logs; logs cannot be altered or deleted even by Super Admins.
* Academic Setup: Full permission to manage grade levels, subject registries, and school schedules.



### 2. School Admin (Principal / Vice-Principal / Registrar)

* **Scope:** School-wide operational management.
* **Responsibilities:** Student enrollment, teacher course allocations, academic calendar oversight, report card verification, grade locking.
* **Permissions:**
* User Management: Create and update Teacher, Student, and Parent accounts; assign students to class sections; assign form teachers.
* Class & Subject Allocation: Create class sections and bind subject teachers to class-subjects.
* Gradebook Oversight: Read all grades across all subjects; override locked grades with required audit logging; lock and unlock academic terms.
* Attendance Management: Review school-wide attendance trends; modify incorrect historical attendance marks upon verification.
* Fee Oversight: View fee structures and billing status for academic tracking.



### 3. Subject Teacher / Educator

* **Scope:** strictly bounded by assigned Class-Subject Allocations.
* **Responsibilities:** Daily attendance taking for assigned classes, gradebook management, assignment submission, term report comments.
* **Permissions:**
* Class Roster: Read student details only for students currently enrolled in their assigned class-subjects.
* Grade Submission: Create and update grades *only* for the specific class-subjects assigned to them, and *only* while the active academic term is unlocked.
* Attendance Recording: Create and edit daily/period attendance records for their assigned classes.
* Announcements: Create class-level announcements visible to assigned students and their parents.
* Restricted Actions: Cannot view grades of students in subjects taught by other teachers; cannot modify grades after a term is locked by an Admin.



### 4. Form Teacher (Homeroom Teacher)

* **Scope:** Bounded to their assigned Class Section as a whole.
* **Responsibilities:** Holistic oversight of a specific section's overall performance, daily homeroom attendance, report card remark compilation.
* **Permissions:**
* All Subject Teacher permissions for their own subjects.
* Class Section Overview: Read-only access to all grades across *all subjects* for students in their assigned homeroom class.
* Homeroom Attendance: Override or compile daily master attendance for their homeroom section.
* Report Cards: Enter general homeroom teacher remarks on term report cards for their class section.



### 5. Student

* **Scope:** Strictly self-bound (`user_id == self.id`).
* **Responsibilities:** View personal academic progress, submit digital coursework, view timetables and announcements.
* **Permissions:**
* Academic Performance: Read-only access to personal grades, assessment breakdowns, and term report cards.
* Attendance: Read-only access to personal attendance history.
* Class Information: Read access to enrolled class timetable, assigned subject lists, and class announcements.
* Restricted Actions: Cannot view grades or records of any other student; cannot alter any submitted marks or official logs.



### 6. Parent / Legal Guardian

* **Scope:** Strictly bound to linked children (`student_id IN (parent.linked_students)`).
* **Responsibilities:** Monitor children's academic standing, track attendance punctuality, review fee statements, communicate with teachers.
* **Permissions:**
* Academic Performance: Read-only access to grades, term report cards, and teacher comments for all explicitly linked children.
* Context Switching: Ability to toggle between multiple linked children within the same portal interface.
* Attendance Tracking: Real-time read-only access to historical and daily attendance statuses for linked children.
* Billing & Payments: Read access to fee breakdown, payment schedules, outstanding balances, and receipt histories for linked children.
* Restricted Actions: Cannot view information of unlinked students; cannot alter academic or administrative data.



### 7. Bursar / Finance Officer

* **Scope:** Financial domain across the entire school.
* **Responsibilities:** Fee collection, invoice generation, payment processing, financial holds management.
* **Permissions:**
* Financial Management: Full create, read, update access to student fee structures, ledger balances, invoices, and payment receipts.
* Student Status: Read-only access to student enrollment lists to verify active billing statuses.
* Restricted Actions: No access to grade submission, academic curriculum setup, or teaching assignments.



---

## End-to-End Operational Workflows & Edge Cases

### 1. The Grade Life-Cycle & Audit Guardrails

The system uses a four-stage grading workflow to protect data integrity:

1. **Draft Stage:** The Subject Teacher enters marks into their digital gradebook. Marks can be updated freely during this active period.
2. **Submission Stage:** The teacher submits the grades for a term. The system validates that all scores fall within valid minimum and maximum boundaries and calculating percentages automatically.
3. **Admin Review & Term Locking:** Once the term ends, the School Admin reviews class averages and locks the term. The moment a term is locked, the system automatically revokes write permissions for Subject Teachers across all endpoints associated with that term.
4. **Post-Lock Corrections (Audit Escalation):** If a grade calculation error is identified after term locking, a teacher cannot change it directly. A change request must be submitted to a School Admin. When an Admin overrides a locked grade, the system forces an audit entry: saving the user ID of the admin, timestamp, old score, new score, and a required text justification.

### 2. Multi-Child Parent Context & Horizontal Security

A major security risk in school portals is horizontal privilege escalation—where a parent manually tampers with API parameters or web links to access another family's child data.

To resolve this, every request originating from a Parent account passes through an authorization filter:

* The system retrieves the logged-in parent's identity.
* It verifies against the parent-student mapping table whether the target student ID is genuinely linked to that parent account.
* If a match exists, access is granted; if not, the system returns a access denied status, completely shielding unauthorized student data.

### 3. Academic Year Transition & Student Advancement

At the end of an academic year, the system executes a bulk transition process:

* Historical data (all grades, class assignments, and attendance records) for the ending year is marked static and archived under that academic year ID.
* Active student profiles are advanced to the next sequence order in the Grade Level table (e.g., Grade 9 students move to Grade 10).
* Students are unassigned from their old class sections, creating a clean slate for Admins to build new class section rosters and assign new teacher allocations for the upcoming academic year.
* Alumni or graduating classes are marked with an inactive/graduated status, preserving their records for historical transcript requests while revoking active student portal features.

---