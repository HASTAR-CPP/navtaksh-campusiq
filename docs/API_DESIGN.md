# RESTful API Specification (v1)
## AI-Powered School Safety, Attendance & Campus Intelligence Platform

---

### 1. API Architecture & Standards

- **Base URL**: `/api/v1/`
- **Protocol**: HTTPS / HTTP/2
- **Data Format**: `application/json` (UTF-8)
- **Authentication**: Bearer JWT (`Authorization: Bearer <access_token>`)
- **Pagination Scheme**: Standard page-number pagination:
  ```json
  {
    "count": 1420,
    "next": "/api/v1/students/?page=3",
    "previous": "/api/v1/students/?page=1",
    "results": [...]
  }
  ```
- **Error Response Standard**:
  ```json
  {
    "error_code": "RESOURCE_CONFLICT",
    "message": "Timetable slot collision detected for Teacher EMP-T-0104 at Period 2.",
    "field_errors": {
      "teacher_id": ["Teacher is already allocated to Room B-101 in this period."]
    },
    "timestamp": "2026-10-05T09:30:00Z"
  }
  ```

---

### 2. Authentication & Authorization (`/api/v1/auth/`)

#### 2.1. User Login
- **Endpoint**: `POST /api/v1/auth/login/`
- **Request Body**:
```json
{
  "username": "sarah.jenkins@school.edu",
  "password": "SecurePassword123!"
}
```
- **Response** (`200 OK`):
```json
{
  "access_token": "eyJhbGciOi...",
  "refresh_token": "eyJhbGciOi...",
  "expires_in": 900,
  "user": {
    "id": "c71a39f6-2db1-4e46-95a3-0a7ef2049d9a",
    "username": "sarah.jenkins@school.edu",
    "full_name": "Dr. Sarah Jenkins",
    "roles": ["TEACHER"],
    "permissions": ["attendance:view", "attendance:mark", "attendance:override"]
  }
}
```

#### 2.2. Token Refresh
- **Endpoint**: `POST /api/v1/auth/refresh/`
- **Request Body**: `{"refresh_token": "eyJhbGciOi..."}`
- **Response** (`200 OK`): `{"access_token": "eyJhbGciOi...", "expires_in": 900}`

#### 2.3. User Logout
- **Endpoint**: `POST /api/v1/auth/logout/`
- **Request Body**: `{"refresh_token": "eyJhbGciOi..."}`
- **Response** (`204 No Content`)

#### 2.4. Current User Profile
- **Endpoint**: `GET /api/v1/auth/me/`
- **Response** (`200 OK`): User details, active roles, assigned classroom IDs.

---

### 3. Students & Academics (`/api/v1/students/`, `/api/v1/academics/`)

#### 3.1. List Students
- **Endpoint**: `GET /api/v1/students/`
- **Query Parameters**:
  - `class_level_id` (int): Filter by grade.
  - `section_id` (int): Filter by section.
  - `status` (string): `ENROLLED`, `SUSPENDED`, `ALUMNI`.
  - `search` (string): Search admission number, student ID, or name.
- **Response** (`200 OK`): Paginated student records.

#### 3.2. Register Student
- **Endpoint**: `POST /api/v1/students/`
- **Permissions**: `ADMIN`, `SUPER_ADMIN`
- **Request Body**:
```json
{
  "user": {
    "first_name": "Alexander",
    "last_name": "Hayes",
    "email": "alex.hayes@student.school.edu"
  },
  "admission_number": "ADM-2026-0812",
  "student_id": "STU-2026-0042",
  "date_of_birth": "2010-04-15",
  "gender": "MALE",
  "blood_group": "O+",
  "admission_date": "2026-08-01",
  "emergency_contact_phone": "+1-555-0199",
  "enrollment": {
    "academic_year_id": 1,
    "section_id": 4,
    "roll_number": 12
  }
}
```
- **Response** (`201 Created`)

#### 3.3. Student Details & Update
- **Endpoint**: `GET /api/v1/students/{id}/`
- **Endpoint**: `PATCH /api/v1/students/{id}/`

---

### 4. Master Timetable (`/api/v1/timetable/`)

#### 4.1. Query Timetable
- **Endpoint**: `GET /api/v1/timetable/`
- **Query Parameters**:
  - `section_id`: Retrieve schedule for a specific section.
  - `teacher_id`: Retrieve faculty schedule.
  - `room_id`: Retrieve classroom occupancy.
  - `day_of_week`: `0` (Monday) to `6` (Sunday).
- **Response** (`200 OK`): Array of schedule slots with period times, teacher, subject, room.

#### 4.2. Create Schedule Slot
- **Endpoint**: `POST /api/v1/timetable/`
- **Permissions**: `ADMIN`, `SUPER_ADMIN`
- **Request Body**:
```json
{
  "academic_year_id": 1,
  "section_id": 4,
  "subject_id": 8,
  "teacher_id": "e28405b5-7489-4089-9810-7463282b09a4",
  "room_id": 12,
  "period_id": 2,
  "day_of_week": 0
}
```
- **Validation**: Enforces room, teacher, and section conflict prevention before committing.
- **Response** (`201 Created` or `409 Conflict`)

#### 4.3. Active Context Resolver (Helper for AI/Teacher)
- **Endpoint**: `GET /api/v1/timetable/active-context/?camera_id=CAM-B204-01`
- **Response** (`200 OK`):
```json
{
  "status": "ACTIVE_SESSION",
  "room": {"id": 12, "room_number": "B-204"},
  "period": {"id": 2, "period_number": 2, "start_time": "09:20", "end_time": "10:05"},
  "class_level": {"id": 2, "name": "Grade 10"},
  "section": {"id": 4, "name": "Section A"},
  "subject": {"id": 8, "name": "Mathematics", "code": "MATH-101"},
  "teacher": {"id": "e28405b5...", "name": "Dr. Sarah Jenkins"}
}
```

---

### 5. Attendance & AI Ingestion (`/api/v1/attendance/`, `/api/v1/ai/`)

#### 5.1. List Attendance Records
- **Endpoint**: `GET /api/v1/attendance/`
- **Query Parameters**:
  - `section_id` (int)
  - `date` (YYYY-MM-DD)
  - `timetable_slot_id` (int)
  - `status` (`PRESENT`, `ABSENT`, `LATE`, `EXCUSED_LEAVE`)
- **Response** (`200 OK`): Roster with status, source (`AI_AUTOMATED` vs `MANUAL`), and timestamp.

#### 5.2. Teacher Manual Attendance Override
- **Endpoint**: `PATCH /api/v1/attendance/{id}/`
- **Permissions**: `TEACHER`, `ADMIN`
- **Request Body**:
```json
{
  "status": "PRESENT",
  "override_reason": "Student entered 5 minutes late due to school bus delay; verified manually."
}
```
- **Response** (`200 OK`): Updates record and writes an immutable entry to `AuditLog`.

#### 5.3. AI Attendance Event Ingestion (Microservice Webhook)
- **Endpoint**: `POST /api/v1/ai/attendance-events/`
- **Permissions**: Machine token (`AI_SERVICE_ROLE`)
- **Request Body**:
```json
{
  "event_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "camera_id": "CAM-B204-01",
  "timestamp": "2026-10-05T09:25:31.420Z",
  "track_id": 4821,
  "candidate_student_id": "STU-2026-0042",
  "similarity_score": 0.842,
  "second_best_score": 0.412,
  "consecutive_hits": 18,
  "total_track_frames": 22,
  "quality_score": 0.94,
  "face_crop_hash": "a1b2c3d4e5f6789012345678abcdef01",
  "bounding_box": {"x_min": 420, "y_min": 180, "x_max": 560, "y_max": 340}
}
```
- **Response** (`200 OK`):
```json
{
  "status": "PROCESSED",
  "decision": "ACCEPTED",
  "attendance_record_id": 10582,
  "recorded_status": "PRESENT"
}
```

#### 5.4. AI Review Queue
- **Endpoint**: `GET /api/v1/ai/review-queue/`
- **Query Parameters**: `status=PENDING`, `section_id=4`
- **Response** (`200 OK`): List of ambiguous candidate events with thumbnail URLs and similarity scores.

#### 5.5. Resolve Review Queue Item
- **Endpoint**: `POST /api/v1/ai/review-queue/{id}/resolve/`
- **Request Body**:
```json
{
  "resolution": "APPROVE_AS_STUDENT",
  "student_id": "STU-2026-0042",
  "reason": "Class teacher visual confirmation."
}
```
- **Response** (`200 OK`)

#### 5.6. Face Enrollment Photo Upload
- **Endpoint**: `POST /api/v1/ai/enrollment/`
- **Permissions**: `ADMIN`
- **Content-Type**: `multipart/form-data`
- **Fields**:
  - `student_id`: UUID
  - `images`: Array of 3 to 5 images (frontal, left 15°, right 15°)
- **Response** (`201 Created`):
```json
{
  "enrollment_id": "3c983421-12ef-410a-9812-984b23d91012",
  "samples_processed": 3,
  "average_quality_score": 0.92,
  "status": "ACTIVE"
}
```

---

### 6. Cameras & Surveillance (`/api/v1/cameras/`, `/api/v1/surveillance/`)

#### 6.1. List & Register Cameras
- **Endpoint**: `GET /api/v1/cameras/`
- **Endpoint**: `POST /api/v1/cameras/` (RTSP credentials stored encrypted; masked in GET output)

#### 6.2. Camera Heartbeat
- **Endpoint**: `POST /api/v1/cameras/{id}/heartbeat/`
- **Request Body**: `{"fps": 24.8, "dropped_frames": 2, "vram_mb": 1420}`
- **Response** (`204 No Content`)

#### 6.3. Campus Surveillance Incidents
- **Endpoint**: `GET /api/v1/surveillance/events/`
- **Query Parameters**: `severity=HIGH`, `status=NEW`
- **Response** (`200 OK`): Security alerts with snapshots and timestamps.

---

### 7. Fees, Exams, Reports & Audit (`/api/v1/fees/`, `/api/v1/reports/`, etc.)

- `GET /api/v1/fees/invoices/?student_id=...`
- `POST /api/v1/fees/payments/`
- `GET /api/v1/exams/`
- `GET /api/v1/reports/attendance/?class_level_id=2&start_date=2026-09-01&end_date=2026-09-30`
- `GET /api/v1/audit/logs/?entity_type=AttendanceRecord&user_id=...`
