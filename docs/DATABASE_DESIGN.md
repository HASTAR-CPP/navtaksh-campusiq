# Database Schema & Relational Design Specification
## PostgreSQL 16 Normalized Relational System of Record

---

### 1. Relational Design Principles

1. **Strict 3NF Normalization**: Redundant strings and unnormalized attributes are strictly partitioned into normalized relational entities with clean Foreign Key constraints.
2. **Data Integrity & Invariants**: Enforce domain invariants via PostgreSQL database-level `CHECK` constraints, composite `UNIQUE` constraints, and foreign key referential integrity (`PROTECT`, `RESTRICT`, `CASCADE`).
3. **Partitioning & Indexing Strategy**: High-volume telemetry tables (`recognition_events`, `attendance_events`, `audit_logs`) use time-based composite B-Tree indexes and are designed for range partitioning by month.
4. **Soft Deletion**: Sensitive operational entities (`Student`, `Teacher`, `AttendanceRecord`) implement soft deletion (`is_deleted`, `deleted_at`) with partial unique indexes to preserve historical integrity.
5. **Separation of Perception from Official Truth**: Visual recognition logs (`recognition_events`, `attendance_events`) remain completely decoupled from official academic attendance records (`attendance_records`).

---

### 2. Entity Relationship Overview

```
 [User] ───────────────┬─────────────────┬─────────────────┐
   │ 1:1               │ 1:1             │ 1:1             │ 1:1
   ▼                   ▼                 ▼                 ▼
[StudentProfile]   [Guardian]       [TeacherProfile]   [StaffProfile]
   │                   │                 │
   ├───────┐           │ (Many:Many)     ├─────────────────┐
   │       ▼           ▼                 │                 │
   │   [StudentGuardianMap]              ▼                 │
   │                               [TimetableSlot] ◄───────┤
   │                                     │                 │
   ▼                                     ▼                 ▼
[Enrollment] ◄── [ClassSection] ───► [Period]          [Subject]
                       ▲                 ▲
                       │                 │
                 [AcademicYear]       [Room] ◄──────── [Camera]
                                         ▲                │
                                         │                ▼
                             [AttendanceRecord]   [AttendanceEvent]
                                                          ▲
                                                          │
                                                  [RecognitionEvent]
```

---

### 3. Detailed Data Dictionaries & DDL Specifications

#### 3.1. Authentication & RBAC (`apps/users`)

```sql
-- Custom User Table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(150) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone_number VARCHAR(20),
    is_active BOOLEAN DEFAULT TRUE NOT NULL,
    is_staff BOOLEAN DEFAULT FALSE NOT NULL,
    is_superuser BOOLEAN DEFAULT FALSE NOT NULL,
    last_login TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- Roles (e.g., SUPER_ADMIN, PRINCIPAL, ADMIN, TEACHER, ACCOUNTANT, SECURITY, STUDENT, PARENT)
CREATE TABLE roles (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- Permissions
CREATE TABLE permissions (
    id SERIAL PRIMARY KEY,
    code VARCHAR(100) UNIQUE NOT NULL, -- e.g., 'attendance:override', 'fee:waive'
    name VARCHAR(150) NOT NULL,
    module VARCHAR(50) NOT NULL,
    description TEXT
);

-- User-Role Mapping
CREATE TABLE user_roles (
    id SERIAL PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id INT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    assigned_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    UNIQUE (user_id, role_id)
);

-- Role-Permission Mapping
CREATE TABLE role_permissions (
    id SERIAL PRIMARY KEY,
    role_id INT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id INT NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    UNIQUE (role_id, permission_id)
);
```

#### 3.2. Academics & Facilities (`apps/academics`)

```sql
-- Academic Years
CREATE TABLE academic_years (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL, -- e.g., '2026-2027'
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    is_active BOOLEAN DEFAULT FALSE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    CHECK (end_date > start_date)
);

-- Physical Rooms & Classrooms
CREATE TABLE rooms (
    id SERIAL PRIMARY KEY,
    room_number VARCHAR(50) UNIQUE NOT NULL, -- e.g., 'B-204'
    building VARCHAR(100) NOT NULL,
    floor INT NOT NULL,
    room_type VARCHAR(50) DEFAULT 'CLASSROOM' NOT NULL, -- 'CLASSROOM', 'LAB', 'LIBRARY', 'HALL'
    seating_capacity INT NOT NULL CHECK (seating_capacity > 0),
    is_active BOOLEAN DEFAULT TRUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- Class Levels (Grades)
CREATE TABLE class_levels (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL, -- e.g., 'Grade 10'
    numeric_order INT NOT NULL,
    description TEXT
);

-- Sections
CREATE TABLE sections (
    id SERIAL PRIMARY KEY,
    class_level_id INT NOT NULL REFERENCES class_levels(id) ON DELETE RESTRICT,
    name VARCHAR(50) NOT NULL, -- e.g., 'A', 'B', 'Omega'
    max_capacity INT DEFAULT 40 NOT NULL,
    default_room_id INT REFERENCES rooms(id) ON DELETE SET NULL,
    UNIQUE (class_level_id, name)
);

-- Subjects
CREATE TABLE subjects (
    id SERIAL PRIMARY KEY,
    code VARCHAR(50) UNIQUE NOT NULL, -- e.g., 'MATH-101'
    name VARCHAR(150) NOT NULL,
    subject_type VARCHAR(30) DEFAULT 'THEORY' NOT NULL, -- 'THEORY', 'PRACTICAL', 'ELECTIVE'
    credits DECIMAL(3, 1) DEFAULT 1.0 NOT NULL
);
```

#### 3.3. Profiles: Students, Guardians, Teachers, Staff (`apps/students`, etc.)

```sql
-- Student Profiles
CREATE TABLE students (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID UNIQUE NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    admission_number VARCHAR(50) UNIQUE NOT NULL, -- e.g., 'ADM-2026-0812'
    student_id VARCHAR(50) UNIQUE NOT NULL,        -- e.g., 'STU-2026-0042'
    date_of_birth DATE NOT NULL,
    gender VARCHAR(20) NOT NULL,
    blood_group VARCHAR(10),
    admission_date DATE NOT NULL,
    status VARCHAR(30) DEFAULT 'ENROLLED' NOT NULL, -- 'ENROLLED', 'SUSPENDED', 'TRANSFERRED', 'ALUMNI'
    emergency_contact_phone VARCHAR(20) NOT NULL,
    medical_notes TEXT,
    is_deleted BOOLEAN DEFAULT FALSE NOT NULL,
    deleted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- Student Academic Enrollments (Annual class placement)
CREATE TABLE enrollments (
    id SERIAL PRIMARY KEY,
    student_id UUID NOT NULL REFERENCES students(id) ON DELETE RESTRICT,
    academic_year_id INT NOT NULL REFERENCES academic_years(id) ON DELETE RESTRICT,
    section_id INT NOT NULL REFERENCES sections(id) ON DELETE RESTRICT,
    roll_number INT NOT NULL,
    enrolled_at DATE DEFAULT CURRENT_DATE NOT NULL,
    is_active BOOLEAN DEFAULT TRUE NOT NULL,
    UNIQUE (academic_year_id, section_id, roll_number),
    UNIQUE (academic_year_id, student_id)
);

-- Guardian Profiles
CREATE TABLE guardians (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID UNIQUE NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    primary_phone VARCHAR(20) NOT NULL,
    secondary_phone VARCHAR(20),
    relationship_type VARCHAR(30) NOT NULL, -- 'FATHER', 'MOTHER', 'LEGAL_GUARDIAN'
    occupation VARCHAR(100),
    address TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- Student-Guardian Mapping
CREATE TABLE student_guardians (
    id SERIAL PRIMARY KEY,
    student_id UUID NOT NULL REFERENCES students(id) ON DELETE CASCADE,
    guardian_id UUID NOT NULL REFERENCES guardians(id) ON DELETE CASCADE,
    is_primary_contact BOOLEAN DEFAULT FALSE NOT NULL,
    is_authorized_pickup BOOLEAN DEFAULT TRUE NOT NULL,
    receives_billing BOOLEAN DEFAULT FALSE NOT NULL,
    receives_sms BOOLEAN DEFAULT TRUE NOT NULL,
    UNIQUE (student_id, guardian_id)
);

-- Teacher Profiles
CREATE TABLE teachers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID UNIQUE NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    employee_id VARCHAR(50) UNIQUE NOT NULL, -- e.g., 'EMP-T-0104'
    qualification VARCHAR(255) NOT NULL,
    specialization VARCHAR(255),
    hire_date DATE NOT NULL,
    status VARCHAR(30) DEFAULT 'ACTIVE' NOT NULL, -- 'ACTIVE', 'ON_LEAVE', 'RESIGNED'
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- Teacher-Subject Mapping
CREATE TABLE teacher_subjects (
    id SERIAL PRIMARY KEY,
    teacher_id UUID NOT NULL REFERENCES teachers(id) ON DELETE CASCADE,
    subject_id INT NOT NULL REFERENCES subjects(id) ON DELETE CASCADE,
    UNIQUE (teacher_id, subject_id)
);

-- Staff Profiles
CREATE TABLE staff (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID UNIQUE NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    employee_id VARCHAR(50) UNIQUE NOT NULL,
    department VARCHAR(100) NOT NULL,
    designation VARCHAR(100) NOT NULL,
    shift_start TIME NOT NULL,
    shift_end TIME NOT NULL,
    is_active BOOLEAN DEFAULT TRUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);
```

#### 3.4. Master Timetable (`apps/timetable`)

```sql
-- Periods Definition
CREATE TABLE periods (
    id SERIAL PRIMARY KEY,
    period_number INT NOT NULL,
    name VARCHAR(50) NOT NULL, -- e.g., 'Period 1', 'Lunch Break'
    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    is_break BOOLEAN DEFAULT FALSE NOT NULL,
    is_active BOOLEAN DEFAULT TRUE NOT NULL,
    CHECK (end_time > start_time),
    UNIQUE (period_number)
);

-- Timetable Slots (The core schedule unit)
CREATE TABLE timetable_slots (
    id SERIAL PRIMARY KEY,
    academic_year_id INT NOT NULL REFERENCES academic_years(id) ON DELETE RESTRICT,
    section_id INT NOT NULL REFERENCES sections(id) ON DELETE RESTRICT,
    subject_id INT NOT NULL REFERENCES subjects(id) ON DELETE RESTRICT,
    teacher_id UUID NOT NULL REFERENCES teachers(id) ON DELETE RESTRICT,
    room_id INT NOT NULL REFERENCES rooms(id) ON DELETE RESTRICT,
    period_id INT NOT NULL REFERENCES periods(id) ON DELETE RESTRICT,
    day_of_week INT NOT NULL CHECK (day_of_week BETWEEN 0 AND 6), -- 0=Monday, 6=Sunday
    is_active BOOLEAN DEFAULT TRUE NOT NULL,
    -- Prevent room scheduling collisions
    CONSTRAINT uq_room_schedule UNIQUE (academic_year_id, room_id, day_of_week, period_id),
    -- Prevent teacher scheduling collisions
    CONSTRAINT uq_teacher_schedule UNIQUE (academic_year_id, teacher_id, day_of_week, period_id),
    -- Prevent section scheduling collisions
    CONSTRAINT uq_section_schedule UNIQUE (academic_year_id, section_id, day_of_week, period_id)
);

-- Timetable Substitutions
CREATE TABLE timetable_substitutions (
    id SERIAL PRIMARY KEY,
    timetable_slot_id INT NOT NULL REFERENCES timetable_slots(id) ON DELETE CASCADE,
    substitute_teacher_id UUID NOT NULL REFERENCES teachers(id) ON DELETE RESTRICT,
    substitution_date DATE NOT NULL,
    reason TEXT NOT NULL,
    created_by UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    UNIQUE (timetable_slot_id, substitution_date)
);
```

#### 3.5. Biometrics & Camera Infrastructure (`apps/cameras`, `apps/ai_events`)

```sql
-- Physical Camera Devices
CREATE TABLE cameras (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    camera_id VARCHAR(50) UNIQUE NOT NULL, -- e.g., 'CAM-B204-01'
    name VARCHAR(150) NOT NULL,
    room_id INT REFERENCES rooms(id) ON DELETE SET NULL,
    zone VARCHAR(50) DEFAULT 'CLASSROOM' NOT NULL, -- 'CLASSROOM', 'GATE', 'CORRIDOR', 'PLAYGROUND'
    ip_address INET,
    rtsp_url_encrypted TEXT NOT NULL,
    is_ptz BOOLEAN DEFAULT FALSE NOT NULL,
    fps INT DEFAULT 25 NOT NULL,
    resolution VARCHAR(20) DEFAULT '1920x1080' NOT NULL,
    status VARCHAR(30) DEFAULT 'ONLINE' NOT NULL, -- 'ONLINE', 'OFFLINE', 'ERROR'
    last_heartbeat TIMESTAMPTZ,
    is_active BOOLEAN DEFAULT TRUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- Student Face Enrollment Records
CREATE TABLE face_enrollments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id UUID NOT NULL REFERENCES students(id) ON DELETE CASCADE,
    sample_count INT DEFAULT 0 NOT NULL,
    quality_score DECIMAL(4, 3) NOT NULL,
    status VARCHAR(30) DEFAULT 'ACTIVE' NOT NULL, -- 'PENDING', 'ACTIVE', 'REVOKED'
    enrolled_by UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- Face Feature Embeddings Metadata (Vectors indexed in FAISS; metadata stored here)
CREATE TABLE face_embeddings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    face_enrollment_id UUID NOT NULL REFERENCES face_enrollments(id) ON DELETE CASCADE,
    student_id UUID NOT NULL REFERENCES students(id) ON DELETE CASCADE,
    faiss_index_id BIGINT UNIQUE NOT NULL,
    model_name VARCHAR(100) NOT NULL,       -- e.g., 'arcface_r100_v1'
    embedding_version INT DEFAULT 1 NOT NULL,
    sample_type VARCHAR(30) NOT NULL,       -- 'FRONTAL', 'LEFT_15', 'RIGHT_15'
    quality_metric DECIMAL(4, 3) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- AI Raw Recognition Events (Per-Frame Detections)
CREATE TABLE recognition_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    camera_id VARCHAR(50) NOT NULL REFERENCES cameras(camera_id) ON DELETE CASCADE,
    timestamp TIMESTAMPTZ NOT NULL,
    track_id BIGINT NOT NULL,
    candidate_student_id UUID REFERENCES students(id) ON DELETE SET NULL,
    similarity_score DECIMAL(5, 4) NOT NULL,
    bounding_box JSONB NOT NULL, -- {x_min, y_min, x_max, y_max}
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);
CREATE INDEX idx_recog_cam_time ON recognition_events(camera_id, timestamp DESC);

-- AI Aggregated Attendance Events (Tracklet Level Proposal)
CREATE TABLE attendance_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id UUID UNIQUE NOT NULL, -- Idempotency key from AI service
    camera_id VARCHAR(50) NOT NULL REFERENCES cameras(camera_id) ON DELETE RESTRICT,
    candidate_student_id UUID NOT NULL REFERENCES students(id) ON DELETE RESTRICT,
    track_id BIGINT NOT NULL,
    event_timestamp TIMESTAMPTZ NOT NULL,
    similarity_score DECIMAL(5, 4) NOT NULL,
    second_best_score DECIMAL(5, 4),
    confidence_margin DECIMAL(5, 4),
    consecutive_hits INT NOT NULL,
    total_track_frames INT NOT NULL,
    quality_score DECIMAL(4, 3) NOT NULL,
    face_crop_hash VARCHAR(64) NOT NULL,
    bounding_box JSONB NOT NULL,
    status VARCHAR(30) DEFAULT 'PENDING' NOT NULL, -- 'PROCESSED', 'REJECTED', 'IN_REVIEW'
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);
CREATE INDEX idx_att_event_lookup ON attendance_events(candidate_student_id, event_timestamp DESC);
```

#### 3.6. Official Attendance & Review Queue (`apps/attendance`, `apps/ai_events`)

```sql
-- Official Verified Attendance Records
CREATE TABLE attendance_records (
    id BIGSERIAL PRIMARY KEY,
    student_id UUID NOT NULL REFERENCES students(id) ON DELETE RESTRICT,
    timetable_slot_id INT NOT NULL REFERENCES timetable_slots(id) ON DELETE RESTRICT,
    attendance_date DATE NOT NULL,
    status VARCHAR(30) NOT NULL, -- 'PRESENT', 'ABSENT', 'LATE', 'EXCUSED_LEAVE'
    source VARCHAR(30) NOT NULL, -- 'AI_AUTOMATED', 'MANUAL_TEACHER', 'ADMIN_OVERRIDE'
    confidence_score DECIMAL(5, 4),
    attendance_event_id UUID REFERENCES attendance_events(id) ON DELETE SET NULL,
    marked_by UUID REFERENCES users(id) ON DELETE SET NULL,
    override_reason TEXT,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    -- Prevent duplicate attendance per student, per schedule slot, per day
    CONSTRAINT uq_student_slot_date UNIQUE (student_id, timetable_slot_id, attendance_date)
);
CREATE INDEX idx_att_student_date ON attendance_records(student_id, attendance_date);
CREATE INDEX idx_att_slot_date ON attendance_records(timetable_slot_id, attendance_date);

-- Teacher AI Review Queue
CREATE TABLE attendance_review_queue (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    attendance_event_id UUID UNIQUE NOT NULL REFERENCES attendance_events(id) ON DELETE CASCADE,
    candidate_student_id UUID REFERENCES students(id) ON DELETE SET NULL,
    timetable_slot_id INT NOT NULL REFERENCES timetable_slots(id) ON DELETE CASCADE,
    event_timestamp TIMESTAMPTZ NOT NULL,
    reason VARCHAR(50) NOT NULL, -- 'LOW_CONFIDENCE', 'AMBIGUOUS_MATCH', 'WRONG_CLASSROOM'
    face_thumbnail_url TEXT NOT NULL,
    status VARCHAR(30) DEFAULT 'PENDING' NOT NULL, -- 'PENDING', 'APPROVED', 'DISCARDED'
    reviewed_by UUID REFERENCES users(id) ON DELETE SET NULL,
    reviewed_at TIMESTAMPTZ,
    teacher_notes TEXT,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);
```

#### 3.7. Fees, Invoices & Payments (`apps/fees`)

```sql
-- Fee Categories
CREATE TABLE fee_categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL, -- 'Tuition Fee', 'Science Lab Fee', 'Bus Transportation'
    description TEXT
);

-- Fee Structure per Academic Year & Class Level
CREATE TABLE fee_structures (
    id SERIAL PRIMARY KEY,
    academic_year_id INT NOT NULL REFERENCES academic_years(id) ON DELETE RESTRICT,
    class_level_id INT NOT NULL REFERENCES class_levels(id) ON DELETE RESTRICT,
    fee_category_id INT NOT NULL REFERENCES fee_categories(id) ON DELETE RESTRICT,
    amount DECIMAL(10, 2) NOT NULL CHECK (amount >= 0),
    billing_frequency VARCHAR(30) DEFAULT 'TERMLY' NOT NULL, -- 'MONTHLY', 'TERMLY', 'ANNUAL'
    due_day_of_month INT DEFAULT 10 NOT NULL,
    UNIQUE (academic_year_id, class_level_id, fee_category_id)
);

-- Student Invoices
CREATE TABLE invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_number VARCHAR(50) UNIQUE NOT NULL, -- e.g., 'INV-2026-00389'
    student_id UUID NOT NULL REFERENCES students(id) ON DELETE RESTRICT,
    academic_year_id INT NOT NULL REFERENCES academic_years(id) ON DELETE RESTRICT,
    total_amount DECIMAL(10, 2) NOT NULL,
    concession_amount DECIMAL(10, 2) DEFAULT 0.00 NOT NULL,
    payable_amount DECIMAL(10, 2) NOT NULL,
    paid_amount DECIMAL(10, 2) DEFAULT 0.00 NOT NULL,
    due_date DATE NOT NULL,
    status VARCHAR(30) DEFAULT 'UNPAID' NOT NULL, -- 'UNPAID', 'PARTIAL', 'PAID', 'OVERDUE'
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- Payments
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_reference VARCHAR(100) UNIQUE NOT NULL, -- Gateway ID / Bank Challan
    invoice_id UUID NOT NULL REFERENCES invoices(id) ON DELETE RESTRICT,
    amount_paid DECIMAL(10, 2) NOT NULL CHECK (amount_paid > 0),
    payment_method VARCHAR(50) NOT NULL, -- 'CASH', 'CREDIT_CARD', 'UPI', 'BANK_TRANSFER'
    transaction_date TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    received_by UUID REFERENCES users(id) ON DELETE SET NULL,
    receipt_number VARCHAR(50) UNIQUE NOT NULL
);
```

#### 3.8. Exams & Gradebook (`apps/exams`)

```sql
-- Exams
CREATE TABLE exams (
    id SERIAL PRIMARY KEY,
    academic_year_id INT NOT NULL REFERENCES academic_years(id) ON DELETE RESTRICT,
    name VARCHAR(150) NOT NULL, -- e.g., 'Mid-Term Examination 2026'
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    is_published BOOLEAN DEFAULT FALSE NOT NULL
);

-- Exam Subject Papers
CREATE TABLE exam_subjects (
    id SERIAL PRIMARY KEY,
    exam_id INT NOT NULL REFERENCES exams(id) ON DELETE CASCADE,
    subject_id INT NOT NULL REFERENCES subjects(id) ON DELETE RESTRICT,
    class_level_id INT NOT NULL REFERENCES class_levels(id) ON DELETE RESTRICT,
    exam_date DATE NOT NULL,
    max_marks DECIMAL(5, 2) NOT NULL,
    pass_marks DECIMAL(5, 2) NOT NULL,
    UNIQUE (exam_id, subject_id, class_level_id)
);

-- Student Marks
CREATE TABLE student_marks (
    id SERIAL PRIMARY KEY,
    exam_subject_id INT NOT NULL REFERENCES exam_subjects(id) ON DELETE RESTRICT,
    student_id UUID NOT NULL REFERENCES students(id) ON DELETE RESTRICT,
    marks_obtained DECIMAL(5, 2) NOT NULL,
    grade VARCHAR(10),
    remarks TEXT,
    graded_by UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL,
    UNIQUE (exam_subject_id, student_id)
);
```

#### 3.9. Surveillance, Notifications, Documents & Audit (`apps/surveillance`, `apps/audit`)

```sql
-- Campus Surveillance Events
CREATE TABLE surveillance_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    camera_id VARCHAR(50) NOT NULL REFERENCES cameras(camera_id) ON DELETE RESTRICT,
    event_type VARCHAR(50) NOT NULL, -- 'UNAUTHORIZED_ZONE_ENTRY', 'LOITERING', 'PERIMETER_BREACH'
    severity VARCHAR(20) DEFAULT 'MEDIUM' NOT NULL, -- 'LOW', 'MEDIUM', 'HIGH', 'CRITICAL'
    timestamp TIMESTAMPTZ NOT NULL,
    track_id BIGINT,
    description TEXT NOT NULL,
    snapshot_url TEXT,
    status VARCHAR(30) DEFAULT 'NEW' NOT NULL, -- 'NEW', 'ACKNOWLEDGED', 'RESOLVED'
    resolved_by UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- School Announcements
CREATE TABLE announcements (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    target_role VARCHAR(50), -- NULL for all, or 'TEACHER', 'STUDENT', 'PARENT'
    target_section_id INT REFERENCES sections(id) ON DELETE SET NULL,
    published_by UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    is_active BOOLEAN DEFAULT TRUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- Immutable Audit Log
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    action VARCHAR(50) NOT NULL, -- 'CREATE', 'UPDATE', 'DELETE', 'OVERRIDE', 'LOGIN'
    entity_type VARCHAR(100) NOT NULL,
    entity_id VARCHAR(100) NOT NULL,
    changes JSONB,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP NOT NULL
);
CREATE INDEX idx_audit_entity ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_user_time ON audit_logs(user_id, created_at DESC);
```
