# School ERP Functional & System Requirements
## AI-Powered School Safety, Attendance & Campus Intelligence Platform

---

### 1. Overview & Institutional Scope

The School ERP serves as the authoritative institutional System of Record (SoR) for all administrative, academic, financial, security, and communication workflows. It operates as a secure, multi-tenant capable, normalized relational application providing granular Role-Based Access Control (RBAC) and clean RESTful APIs.

---

### 2. Role-Based Access Control (RBAC) Matrix

The system enforces strict permission tiers. Every API endpoint and data view validates both role membership and row-level ownership.

| Role | Scope of Access & Permissions |
| :--- | :--- |
| **`SUPER_ADMIN`** | Full institutional access; database configurations, system settings, global audit logs, user provisioning, role assignments, tenant management. |
| **`PRINCIPAL`** | Campus-wide visibility; academic schedules, staff performance, school-wide attendance analytics, fee overviews, safety and security incident oversight, announcement broadcasts. |
| **`ADMIN`** | Operational administration; student admissions, teacher onboarding, class/section assignments, master timetable creation, camera provisioning, review queue escalation. |
| **`TEACHER`** | Classroom operations; view assigned classes and enrolled students, access daily timetable, manage official attendance, review AI-flagged attendance exceptions, submit marks. |
| **`ACCOUNTANT`** | Financial management; fee structures, student invoice generation, payment processing, fee receipts, concession management, overdue balance reporting. |
| **`SECURITY`** | Campus safety operations; camera status monitoring, stream health, surveillance alerts (loitering, unauthorized zone access, unassigned individuals), perimeter incident logging. |
| **`STUDENT`** | Self-service portal; view personal academic timetable, personal attendance history, examination marks, fee status, and campus notices. Strict read-only scope. |
| **`PARENT`** | Guardian portal; view linked child(ren) attendance, academic performance, fee invoices, direct payment portal, timetable, and school announcements. |

---

### 3. Detailed Functional Modules

#### 3.1. User & Identity Management (`apps/users`)
- **Custom User Model**: Primary identifier via institutional email or unique system username. Supports phone number, active status, last login, and multi-factor authentication (MFA) capability.
- **Roles & Permissions**: Django auth groups extended with domain-specific permission codes (e.g., `attendance.override_record`, `fees.waive_late_fee`, `camera.view_rtsp`).
- **Session & JWT Security**: Short-lived JWT access tokens ($15$ minutes) paired with rotating refresh tokens ($7$ days), with immediate revocation upon logout.

#### 3.2. Student Information System (`apps/students`)
- **Student Profile**: Unique `student_id` (e.g., `STU-2026-0042`), admission number, first/last names, date of birth, gender, blood group, medical alerts, admission date, active enrollment status (`ENROLLED`, `SUSPENDED`, `ALUMNI`, `TRANSFERRED`).
- **Classroom Enrollment**: History of student placements into `AcademicYear` $\times$ `ClassLevel` $\times$ `Section` with assigned roll numbers.
- **Biometric Linkage**: Reference to student's face enrollment profile and vector gallery ID (strictly zero raw biometric data stored directly in student relational tables).

#### 3.3. Guardian Management (`apps/guardians`)
- **Guardian Profile**: Full name, relationship type (`FATHER`, `MOTHER`, `LEGAL_GUARDIAN`, `EMERGENCY_CONTACT`), occupation, primary phone, secondary phone, email address, national identity ID (masked), physical address.
- **Student-Guardian Mapping**: Many-to-Many relation with attributes: `is_primary_contact`, `is_authorized_pickup`, `receives_billing`, `receives_attendance_sms`.

#### 3.4. Teacher & Faculty Management (`apps/teachers`)
- **Faculty Profile**: Unique `employee_id`, department, designation, qualifications, hire date, status (`ACTIVE`, `ON_LEAVE`, `RESIGNED`).
- **Subject Specializations**: Many-to-Many mapping to `Subject` entities.
- **Class Teacher Assignment**: Official designated class teacher for specific `ClassLevel` and `Section` combinations.

#### 3.5. Staff Management (`apps/staff`)
- **Non-Academic Staff**: Administrative staff, IT staff, accountants, security personnel, bus drivers, and facilities maintenance.
- **Departmental Hierarchy**: Department assignment, shift timings, access badge numbers, emergency contact info.

#### 3.6. Academics Core (`apps/academics`)
- **`AcademicYear`**: Name (e.g., "2026-2027"), start date, end date, is_active flag.
- **`ClassLevel`**: Grade or standard (e.g., "Grade 10", "Grade 11").
- **`Section`**: Class division (e.g., "Section A", "Section B"), max capacity.
- **`Subject`**: Subject title (e.g., "Advanced Mathematics"), code (e.g., "MATH-101"), credit units, subject type (`THEORY`, `PRACTICAL`, `ELECTIVE`).
- **`Room`**: Physical building, floor, room number (e.g., "Room B-204"), room type (`CLASSROOM`, `SCIENCE_LAB`, `AUDITORIUM`, `LIBRARY`), seating capacity.

#### 3.7. Timetable Engine (`apps/timetable`)
- **`Period`**: Defined institutional time intervals with strict start and end times (e.g., Period 1: 08:30 - 09:15, Period 2: 09:20 - 10:05).
- **`TimetableSlot`**: The central relational binding:
  $$\text{AcademicYear} \times \text{ClassLevel} \times \text{Section} \times \text{DayOfWeek} \times \text{Period} \to (\text{Subject}, \text{Teacher}, \text{Room})$$
- **Conflict Prevention Engine**: Database unique constraints and validation logic prevent:
  - Teacher assigned to two rooms during the same period.
  - Room assigned to two classes during the same period.
  - Section scheduled for two subjects during the same period.
- **Substitutions**: Temporary single-day teacher reassignments preserving timetable history.

#### 3.8. Attendance Management (`apps/attendance`)
- **Attendance Granularity**: Supports both Daily Campus Attendance (gate/morning roll call) and Period-Wise Attendance (per subject period).
- **Attendance Record Statuses**: `PRESENT`, `ABSENT`, `LATE`, `EXCUSED_LEAVE`, `HALF_DAY`.
- **Source of Attendance**: Record source tracked explicitly (`MANUAL_TEACHER`, `AI_AUTOMATED`, `ADMIN_OVERRIDE`).
- **Leave Application Workflow**: Parent/student submits leave request $\to$ Teacher/Principal approves $\to$ Auto-excuses attendance records.
- **Override Audit Requirement**: Any manual change to an attendance record requires a non-empty `override_reason` and logs user ID, timestamp, old value, and new value to `AuditLog`.

#### 3.9. AI Event Ingestion & Review Queue (`apps/ai_events`)
- **`RecognitionEvent`**: Raw sensor data from AI inference service. Contains `camera_id`, detected `face_crop_hash`, FAISS candidate ID, cosine similarity, bounding box coordinates, and timestamp.
- **`AttendanceEvent`**: Filtered candidate attendance proposal with tracklet continuity and multi-frame consistency metrics.
- **`ReviewQueueItem`**: Unresolved or ambiguous events requiring teacher attention:
  - Ambiguous identity match (similarity between $0.45$ and $0.65$).
  - Student detected in a classroom where they are not enrolled.
  - Multiple candidate identities within a single track.
- **Resolution Action**: Teacher can `APPROVE_AS_STUDENT(student_id)`, `MARK_UNKNOWN`, or `DISCARD`.

#### 3.10. Camera & Sensor Infrastructure (`apps/cameras`)
- **Camera Registry**: `camera_id` (e.g., `CAM-B204-01`), human-readable name, hardware manufacturer, model, MAC address, internal IP address, RTSP endpoint URL template.
- **Spatial Mapping**: Direct foreign key to `Room` or `CameraZone` (`ENTRANCE_GATE`, `CORRIDOR`, `PLAYGROUND`, `PERIMETER`).
- **Security Credentials**: Camera RTSP credentials encrypted at rest using Fernet/AES-256 and never exposed through public REST APIs.
- **Health & Telemetry**: Ping latency, stream FPS, dropped frames, last heartbeat timestamp, active status.

#### 3.11. Campus Surveillance & Incident Hub (`apps/surveillance`)
- **Surveillance Events**: Logged safety incidents:
  - `UNAUTHORIZED_ZONE_ENTRY`: Detection in restricted areas after hours.
  - `UNKNOWN_PERSON_LOITERING`: Persistent unassigned face tracklet.
  - `PERIMETER_BREACH`: Outer fence motion/person detection.
- **Incident Escalation**: Real-time push notification to Security Guard dashboard and Principal console.

#### 3.12. Examination & Gradebook (`apps/exams`)
- **`Exam`**: Exam series (e.g., "Mid-Term Examination 2026"), academic year, start/end dates.
- **`ExamSubject`**: Max marks, pass marks, weightage percentage.
- **`StudentMark`**: Obtained marks, grade, teacher comments, moderation status.
- **Report Card Generation**: Automated GPA calculation, percentage aggregation, printable PDF transcripts.

#### 3.13. Fee Management & Invoicing (`apps/fees`)
- **`FeeCategory`**: Tuition, Laboratory, Transportation, Library, Annual Facility.
- **`FeeStructure`**: Defined amounts per `AcademicYear` and `ClassLevel`.
- **`Invoice`**: Generated per student per billing cycle (Term / Month / Annual). Tracks total amount, concessions, paid amount, due date, payment status (`PAID`, `PARTIAL`, `UNPAID`, `OVERDUE`).
- **`PaymentTransaction`**: Transaction ID, gateway reference, amount paid, payment method (`CASH`, `CARD`, `ONLINE_UPI`, `BANK_TRANSFER`), timestamp, accountant receipt number.

#### 3.14. Communication & Notices (`apps/communication`)
- **`Announcement`**: School-wide or targeted bulletin (target roles, specific classes/sections).
- **`Notification`**: Direct user-level alerts (in-app, SMS, email) with read receipt tracking.

#### 3.15. Document Management (`apps/documents`)
- **Document Storage**: Secure file storage via MinIO/S3 object abstraction.
- **Document Types**: Birth certificates, transfer certificates, medical immunizations, previous academic transcripts, national ID proofs.

#### 3.16. Immutable Audit Trail (`apps/audit`)
- **`AuditLog`**: Records all state modifications:
  - `user`: Initiating user.
  - `action`: `CREATE`, `UPDATE`, `DELETE`, `OVERRIDE`, `LOGIN`, `LOGOUT`.
  - `entity_type`: String name of affected model.
  - `entity_id`: Primary key of affected record.
  - `changes`: JSON diff of modified attributes `{"old": ..., "new": ...}`.
  - `ip_address`: Client IP.
  - `user_agent`: Client user agent.
  - `timestamp`: UTC ISO-8601 timestamp.
  - Tamper-evident: Append-only table with no update/delete permissions granted to any application user.
