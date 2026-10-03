# Development Roadmap & Implementation Phasing
## AI-Powered School Safety, Attendance & Campus Intelligence Platform

---

### 1. Phased Engineering Strategy

To guarantee production quality, zero regression, and disciplined separation between the core School ERP and the AI vision service, development follows a structured 13-phase engineering roadmap:

```
[ Phase 0: Inspection & Discovery ] ──► [ Phase 1: Architecture Specs (COMPLETED) ]
                                                        │
                                                        ▼
[ Phase 3: RBAC & Auth ] ◄── [ Phase 2: Django & DB Foundation ]
            │
            ▼
[ Phase 4: Academics & Profiles ] ──► [ Phase 5: Timetable Engine ]
                                                        │
                                                        ▼
[ Phase 7: AI Event APIs ] ◄── [ Phase 6: Attendance Core ]
            │
            ▼
[ Phase 8: FastAPI AI Perception ] ──► [ Phase 9: Attendance Decision Engine ]
                                                        │
                                                        ▼
[ Phase 11: Fees/Exams/Comms ] ◄── [ Phase 10: Frontend Dashboards ]
            │
            ▼
[ Phase 12: Production Hardening, Docker & Observability ]
```

---

### 2. Phase-by-Phase Roadmap & Milestones

#### Phase 0: Repository Inspection & Discovery (Status: Complete)
- **Milestones**:
  - Full inspection of workspace files, git status, and installed runtimes.
  - Verification that workspace is in greenfield state with staged `SYSTEM_BLUEPRINT.md`.
  - Identification of available system runtimes (Python 3.13.2, OpenCV 4.12, NumPy 2.2).
- **Deliverables**: Discovery logs, initial baseline assessment.

#### Phase 1: Architectural Blueprint & Documentation (Status: Complete)
- **Milestones**:
  - Author comprehensive architectural specifications matching enterprise requirements.
  - Define data boundary: `RecognitionEvent` vs `AttendanceEvent` vs `AttendanceRecord`.
  - Specify model abstraction interfaces for modular CV backends.
- **Deliverables**:
  - `docs/ARCHITECTURE.md`
  - `docs/ERP_REQUIREMENTS.md`
  - `docs/AI_ERP_INTEGRATION.md`
  - `docs/DATABASE_DESIGN.md`
  - `docs/API_DESIGN.md`
  - `docs/SECURITY.md`
  - `docs/DEVELOPMENT_ROADMAP.md`

#### Phase 2: Django ERP Foundation & Database Configuration
- **Milestones**:
  - Initialize `backend/erp` project with settings split (`base.py`, `development.py`, `production.py`).
  - Configure PostgreSQL database connection with connection pooling and SSL modes.
  - Setup core abstract model (`BaseModel` with UUID primary keys, created_at, updated_at).
  - Setup Celery, Redis broker, and structured logging handlers.
  - Configure root URL routing, OpenAPI/Swagger documentation (`drf-spectacular`).
- **Deliverables**: Working Django backend skeleton, test database migrations.

#### Phase 3: User Identity, Authentication & Granular RBAC
- **Milestones**:
  - Implement custom `User` model, `Role`, `Permission`, `UserRole`, `RolePermission`.
  - Seed initial institutional roles: `SUPER_ADMIN`, `PRINCIPAL`, `ADMIN`, `TEACHER`, `ACCOUNTANT`, `SECURITY`, `STUDENT`, `PARENT`.
  - Implement JWT authentication with refresh token rotation and blacklist revocation.
  - Implement role and permission authorization decorators and DRF permission classes.
- **Deliverables**: Passing unit tests for JWT login, token refresh, and RBAC permission checks.

#### Phase 4: Academics Structure, Facilities & Institutional Profiles
- **Milestones**:
  - Implement `AcademicYear`, `ClassLevel`, `Section`, `Subject`, `Room`.
  - Implement profile models: `Student`, `Guardian`, `StudentGuardianMap`, `Teacher`, `Staff`.
  - Implement `Enrollment` model linking students to academic years and sections.
  - Implement serializers, viewsets, filters, and pagination for students and teachers.
- **Deliverables**: Complete CRUD APIs for academic directory with audit log integration.

#### Phase 5: Master Timetable Engine & Conflict Detection
- **Milestones**:
  - Implement `Period`, `TimetableSlot`, and `TimetableSubstitution` models.
  - Implement conflict validation logic preventing teacher, room, and section overlap.
  - Implement context resolution API (`/api/v1/timetable/active-context/`) mapping Camera $\to$ Room $\to$ Active Period $\to$ Class/Section/Subject/Teacher.
- **Deliverables**: Passing unit tests for timetable collision prevention and real-time context resolution.

#### Phase 6: Attendance Management Core
- **Milestones**:
  - Implement `AttendanceRecord` supporting daily and period-wise attendance.
  - Implement business rules: duplicate prevention (`uq_student_slot_date`), leave adjustments.
  - Implement manual teacher override endpoint requiring `override_reason` and generating `AuditLog`.
  - Implement attendance summary and percentage aggregation queries.
- **Deliverables**: Attendance CRUD, manual override audit trails, report generation.

#### Phase 7: AI Event Ingestion APIs & Review Queue
- **Milestones**:
  - Implement `RecognitionEvent` and `AttendanceEvent` models in `apps/ai_events`.
  - Implement `AttendanceReviewQueue` for ambiguous or disputed detections.
  - Build endpoints: `POST /api/v1/ai/attendance-events/`, `GET /api/v1/ai/review-queue/`, `POST /api/v1/ai/review-queue/{id}/resolve/`.
  - Implement HMAC signature validation and machine token auth for AI service payloads.
- **Deliverables**: Ingestion endpoints with idempotency validation and review queue workflow.

#### Phase 8: FastAPI AI Computer Vision Service
- **Milestones**:
  - Initialize `ai/ai_service` FastAPI microservice structure.
  - Implement Model Abstraction interfaces: `BaseDetector`, `BaseFaceDetector`, `BaseFaceRecognizer`, `BaseTracker`, `BaseVectorStore`.
  - Implement concrete wrappers: YOLOv8 / RT-DETR, SCRFD / RetinaFace, ArcFace / AdaFace, ByteTrack, FAISS.
  - Implement face quality assessment gatekeeper (Laplacian variance, illumination, head pose, landmark confidence).
  - Implement multi-frame tracklet temporal evidence accumulator.
  - Implement asynchronous event dispatcher client with local SQLite buffer and retry queue.
- **Deliverables**: End-to-end pipeline test with synthetic/sample video streams emitting structured events.

#### Phase 9: End-to-End Attendance Decision Engine
- **Milestones**:
  - Connect FastAPI perception stream to Django ERP context resolver.
  - Implement multi-candidate evaluation, score margin checks, and enrollment verification.
  - Wire decision routing: `ACCEPTED` $\to$ `AttendanceRecord`, `AMBIGUOUS` $\to$ `ReviewQueue`, `UNKNOWN` $\to$ Security Log.
  - Real-time notification dispatch via WebSockets / Redis PubSub.
- **Deliverables**: Full integration test verifying camera frame $\to$ detection $\to$ timetable resolution $\to$ official attendance record.

#### Phase 10: Frontend Dashboards & Teacher Console
- **Milestones**:
  - Setup React / Next.js TypeScript frontend with modern enterprise styling.
  - Build Admin Command Center: Camera status, campus occupancy, real-time event ticker.
  - Build Teacher Console: Live classroom attendance roster, one-tap overrides, AI review queue modal.
  - Build Parent/Student Portal: Attendance history calendar, timetable view, announcements.
  - Build Security Console: Real-time surveillance events, perimeter alerts, unknown person logs.
- **Deliverables**: Interactive, responsive web client integrated with ERP REST & WebSocket APIs.

#### Phase 11: Ancillary ERP Modules (Fees, Exams, Announcements, Documents)
- **Milestones**:
  - Implement fee structures, student invoices, payment receipts (`apps/fees`).
  - Implement exam schedules, grading sheets, GPA calculations (`apps/exams`).
  - Implement announcements and notification dispatch (`apps/communication`).
  - Implement document archival with MinIO / S3 signed URLs (`apps/documents`).
- **Deliverables**: Full ERP feature parity meeting institutional operational requirements.

#### Phase 12: Production Hardening, Docker Orchestration & Observability
- **Milestones**:
  - Finalize multi-stage Dockerfiles (`Dockerfile.erp`, `Dockerfile.ai`, `Dockerfile.frontend`).
  - Create complete `docker-compose.yml` (PostgreSQL, Redis, Django, AI service, Frontend, Nginx, MinIO).
  - Implement Prometheus metrics endpoints (inference latency, camera FPS, queue depths, API response times).
  - Configure automated database backup scripts, SSL termination, and rate-limiting rules.
- **Deliverables**: One-click local orchestration (`docker compose up --build`), deployment documentation, production runbooks.
