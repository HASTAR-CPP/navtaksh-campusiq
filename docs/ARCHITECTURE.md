# System Architecture Specification
## AI-Powered School Safety, Attendance & Campus Intelligence Platform

---

### 1. Architectural Philosophy & Core Axioms

This platform is engineered as a mission-critical, enterprise-grade dual-system:
1. **The School ERP (`backend/erp`) is the authoritative System of Record (SoR).**
   It owns institutional ground truth: identities, roles, academic years, classes, sections, enrollments, master timetables, room assignments, leave records, official attendance logs, fees, and audit trails.
2. **The AI Vision Platform (`ai/ai_service`) is a high-throughput, decoupled perception service.**
   It consumes camera streams (RTSP/ONVIF), executes detection, multi-object tracking, facial feature extraction, and FAISS vector matching.
3. **Strict Separation of Concerns:**
   - The AI vision service **never** writes directly to or modifies ERP database tables.
   - The AI inference pipeline is **never** embedded inside Django processes.
   - Communication between the AI engine and ERP occurs strictly over authenticated, versioned REST APIs and asynchronous message queues (Redis / Celery / WebSockets).
4. **Perception vs. Official Record Boundary:**
   - A raw visual detection is a `RecognitionEvent`.
   - A temporally aggregated and track-consistent candidate is an `AttendanceEvent`.
   - An approved, timetable-reconciled, school-authorized entry is an `AttendanceRecord`.
   - **Never** convert a single raw camera frame into an official attendance entry.

---

### 2. High-Level System Architecture Diagram

```
 +-----------------------------------------------------------------------------------+
 |                             PHYSICAL CAMPUS LAYER                                 |
 |  [ RTSP / ONVIF IP Cameras ]  [ Classroom Cameras ]  [ Gates / Perimeter Cameras ] |
 +-----------------------------------------------------------------------------------+
                                           │
                                           │ RTSP / UDP / TCP Streams
                                           ▼
 +-----------------------------------------------------------------------------------+
 |                       AI COMPUTER VISION INFERENCE SERVICE                        |
 |                         (FastAPI + PyTorch / ONNX / TensorRT)                      |
 |                                                                                   |
 |  [ Stream Ingestion & Decoding (NVDEC / VAAPI / OpenCV GStreamer Ring Buffers) ]  |
 |                                         │                                         |
 |  [ Object & Person Detection ] ──────► [ Multi-Object Tracking (ByteTrack) ]      |
 |   (YOLOv8 / RT-DETR / RF-DETR)                  │                                 |
 |                                                 ▼                                 |
 |  [ Face Detector & Landmarker ] ─────► [ Face Quality & Pose Verification ]       |
 |   (SCRFD / RetinaFace)                          │ (Laplacian blur, Yaw/Pitch/Roll)|
 |                                                 ▼                                 |
 |  [ Feature Extraction / Embedding ] ──► [ Vector Gallery Search (FAISS) ]        |
 |   (ArcFace / AdaFace / SFace 512-d)             │ (Cosine Similarity Metric)      |
 |                                                 ▼                                 |
 |  [ Tracklet-Level Temporal Evidence Accumulator ]                                 |
 |   (N-hits over W-window, Confidence Score, Track Continuity)                      |
 +-----------------------------------------------------------------------------------+
                                           │
                                           │ Authenticated REST / Webhook
                                           │ /api/v1/ai/attendance-events/
                                           ▼
 +-----------------------------------------------------------------------------------+
 |                        CORE SCHOOL ERP (Django + DRF)                             |
 |                                                                                   |
 |  +───────────────────────+   +───────────────────────+   +─────────────────────+  |
 |  |  Event Ingestion API  |──►|   Context Resolver    |──►|  Decision Engine    |  |
 |  |  (Payload Validation, |   |   (Camera -> Room     |   |  (Enrolled Check,   |  |
 |  |   HMAC/Token Auth)    |   |    Time -> Timetable) |   |   Duplicate Check)  |  |
 |  +───────────────────────+   +───────────────────────+   +─────────────────────+  |
 |                                                                     │             |
 |                                    ┌────────────────────────────────┼──────────┐  |
 |                                    ▼                                ▼          ▼  |
 |                         [ Official Attendance ]             [ Review Queue ] [Audit]
 +-----------------------------------------------------------------------------------+
                                           │
                                           │ SQL Transactions & Cache
                                           ▼
 +-----------------------------------------------------------------------------------+
 |                        INFRASTRUCTURE & STORAGE LAYER                             |
 |   [ PostgreSQL 16 (Relational SoR) ]       [ Redis 7 (Broker & Timetable Cache) ] |
 |   [ Celery (Asynchronous Task Queue) ]     [ MinIO / S3 (Encrypted Enrollments) ]  |
 +-----------------------------------------------------------------------------------+
                                           │
                                           │ REST API / JWT / WebSockets
                                           ▼
 +-----------------------------------------------------------------------------------+
 |                          FRONTEND CLIENT PRESENTATION                             |
 |       (Next.js / React 18+ TypeScript, TailwindCSS / Vanilla Enterprise CSS)      |
 |                                                                                   |
 |  [ Admin Command Center ]   [ Teacher Console ]  [ Parent App ]  [ Security Ops ] |
 +-----------------------------------------------------------------------------------+
```

---

### 3. Model Abstraction Layer (Clean Plugin Interfaces)

To avoid vendor or model lock-in and enable rigorous benchmarking without code rewrites, the AI service enforces abstract interfaces:

```
                       +-------------------------+
                       |      BaseDetector       |
                       +-------------------------+
                                    ▲
             ┌──────────────────────┴──────────────────────┐
             │                                             │
+-------------------------+                   +-------------------------+
|     YOLOv8Detector      |                   |     RTDETRDetector      |
+-------------------------+                   +-------------------------+

                       +-------------------------+
                       |    BaseFaceDetector     |
                       +-------------------------+
                                    ▲
             ┌──────────────────────┴──────────────────────┐
             │                                             │
+-------------------------+                   +-------------------------+
|      SCRFDDetector      |                   |   RetinaFaceDetector    |
+-------------------------+                   +-------------------------+

                       +-------------------------+
                       |   BaseFaceRecognizer    |
                       +-------------------------+
                                    ▲
             ┌──────────────────────┴──────────────────────┐
             │                                             │
+-------------------------+                   +-------------------------+
|    ArcFaceRecognizer    |                   |    AdaFaceRecognizer    |
+-------------------------+                   +-------------------------+

                       +-------------------------+
                       |     BaseVectorStore     |
                       +-------------------------+
                                    ▲
             ┌──────────────────────┴──────────────────────┐
             │                                             │
+-------------------------+                   +-------------------------+
|     FAISSVectorStore    |                   |    Qdrant / Milvus      |
+-------------------------+                   +-------------------------+
```

Every module implements strict typed contracts:
- `BaseDetector`: `detect(frame: np.ndarray) -> List[DetectionBox]`
- `BaseTracker`: `update(detections: List[DetectionBox], frame: np.ndarray) -> List[TrackedObject]`
- `BaseFaceDetector`: `detect_faces(frame: np.ndarray) -> List[FaceBoxWithLandmarks]`
- `BaseFaceRecognizer`: `extract_embedding(aligned_face: np.ndarray) -> np.ndarray (512-d float32)`
- `BaseVectorStore`: `search(embedding: np.ndarray, top_k: int) -> List[MatchResult]`

---

### 4. Component Topology & Data Flow

#### 4.1. Edge Video Ingestion
1. Camera streams are ingested via RTSP over TCP with reconnect back-off.
2. Ingestion threads decode frames using hardware-accelerated pipelines into thread-safe shared ring-buffers.
3. Frames are dropped adaptively under load to maintain real-time tracking invariants.

#### 4.2. Perception & Tracking Pipeline
1. **Object Detection**: Detect persons in the frame.
2. **ByteTrack**: Associate detections with existing trajectory tracks (`track_id`).
3. **Face Detection & Landmarking**: For tracked persons whose faces are visible, run SCRFD to obtain bounding box and 5-point facial landmarks.
4. **Face Quality Assessment**:
   - Laplacian variance for blur filter ($\sigma^2 \ge 100$).
   - Brightness histogram validation ($40 \le \mu \le 220$).
   - 3D Head pose estimation ($|\text{yaw}| \le 30^\circ$, $|\text{pitch}| \le 25^\circ$, $|\text{roll}| \le 20^\circ$).
   - Reject frames with severe occlusion or low landmark confidence.
5. **Alignment & Feature Extraction**:
   - Standard 5-point affine transformation to $112 \times 112$ canonical face crops.
   - ArcFace/AdaFace forward pass yielding an L2-normalized 512-dimensional vector.
6. **FAISS Vector Gallery Matching**:
   - Cosine similarity search against registered student/staff vector galleries.
   - Return top candidate identity ID, similarity score ($S$), and second-best candidate margin ($S_1 - S_2$).

#### 4.3. Tracklet Temporal Evidence Aggregator
- A single high-similarity frame is insufficient to mark attendance.
- Detections are accumulated per `track_id` over a sliding time window (e.g., $3.0$ seconds).
- Requires at least $M$ consistent identity matches with mean similarity $\ge \tau_{\text{accept}}$ (e.g., $0.68$).
- Upon satisfying streak conditions, an `AttendanceEvent` is compiled and dispatched to the ERP.

#### 4.4. ERP Context Resolution & Decision Engine
1. **Ingest & Verify**: The ERP receives the `AttendanceEvent` over `/api/v1/ai/attendance-events/`.
2. **Context Resolution**:
   - Map `camera_id` $\to$ `Room` (e.g., Room B-204).
   - Query master timetable for current timestamp:
     `Room` + `Current Time` $\to$ `AcademicYear`, `Class`, `Section`, `Subject`, `Teacher`, `Period`.
3. **Decision Logic**:
   - **Enrolled Check**: Is candidate student enrolled in the resolved `Class` and `Section`?
   - **Schedule Check**: Is the period currently active (within allowable entry window)?
   - **Duplicate Check**: Has attendance already been recorded for this student in this period?
   - **Confidence Routing**:
     - Similarity $\ge \tau_{\text{accept}}$ + Correct Classroom $\to$ `ACCEPTED` (`PRESENT` or `LATE`).
     - Correct Classroom + Similarity between $\tau_{\text{review}}$ and $\tau_{\text{accept}}$ $\to$ `REVIEW_REQUIRED`.
     - Student recognized in a classroom where they are NOT scheduled $\to$ `REVIEW_REQUIRED` (Wrong Class Alert).
     - Similarity $< \tau_{\text{unknown}}$ $\to$ `UNKNOWN`.
4. **State Persistence**:
   - Write `AttendanceRecord` if accepted.
   - Route to `AttendanceReviewQueue` if review required.
   - Always log `AuditLog` entry.
   - Push real-time event via WebSocket / Redis PubSub to Teacher Console & Admin Dashboard.

---

### 5. Target Monorepo Structure

```
animesh-platform/
│
├── backend/
│   └── erp/                          # Django 5.x + DRF Core System of Record
│       ├── manage.py
│       ├── config/                   # Settings, urls, wsgi, asgi, celery
│       │   ├── settings/
│       │   │   ├── base.py
│       │   │   ├── development.py
│       │   │   └── production.py
│       │   ├── urls.py
│       │   └── celery.py
│       ├── apps/                     # Modular ERP Django Applications
│       │   ├── core/                 # Abstract models, base managers, helpers
│       │   ├── users/                # Custom User, Roles, Permissions, Auth
│       │   ├── students/             # Student records, enrollments, profiles
│       │   ├── guardians/            # Parents/Guardians, relationship mappings
│       │   ├── teachers/             # Faculty profiles, qualifications, subjects
│       │   ├── staff/                # Administrative & support staff
│       │   ├── academics/            # AcademicYears, Classes, Sections, Subjects, Rooms
│       │   ├── timetable/            # Periods, TimetableSlots, Allocations
│       │   ├── attendance/           # Official attendance records, daily/period sheets
│       │   ├── ai_events/            # RecognitionEvents, AttendanceEvents, ReviewQueue
│       │   ├── cameras/              # Camera nodes, zones, RTSP stream metadata
│       │   ├── surveillance/         # Campus security events, alerts, incidents
│       │   ├── exams/                # Exams, ExamSubjects, Marks grading
│       │   ├── fees/                 # Fee structures, Invoices, Payments
│       │   ├── communication/        # Announcements, Notifications, Templates
│       │   ├── documents/            # Student/staff document archives
│       │   ├── reports/              # Aggregated academic, attendance, financial reports
│       │   └── audit/                # Immutable AuditLog with before/after diffs
│       └── tests/                    # Enterprise unit, model, rule & API test suite
│
├── ai/
│   └── ai_service/                   # FastAPI Inference & Perception Microservice
│       ├── main.py                   # FastAPI app entry point, lifespans, router mounts
│       ├── config.py                 # Pydantic v2 settings, hardware configs
│       ├── pipelines/                # Vision execution pipelines
│       │   ├── stream_manager.py     # Threaded RTSP capture & ring buffers
│       │   ├── face_pipeline.py      # Detect -> Align -> Quality -> Embed -> Match
│       │   └── temporal_aggregator.py# Tracklet-level evidence streak accumulator
│       ├── models/                   # Concrete model implementations
│       │   ├── base.py               # Abstract base classes (Detector, Recognizer, etc.)
│       │   ├── detectors/            # YOLO, SCRFD, RetinaFace wrappers
│       │   ├── trackers/             # ByteTrack wrapper
│       │   ├── recognizers/          # ArcFace, AdaFace ONNX/PyTorch wrappers
│       │   └── vector_stores/        # FAISS index wrapper with thread-safe updates
│       ├── quality/                  # Face quality: blur, illumination, pose estimation
│       ├── client/                   # Authenticated ERP API client with retry queues
│       └── tests/                    # Synthetic frame tests, pipeline benchmarks
│
├── frontend/                         # Next.js 14+ / React 18+ TypeScript Client
│   ├── src/
│   │   ├── app/                      # Next.js App Router (auth, admin, teacher, parent, security)
│   │   ├── components/               # Modular UI design system components
│   │   ├── hooks/                    # Auth, WebSockets, SWR/React Query hooks
│   │   ├── lib/                      # API client, token managers, formatting utils
│   │   └── types/                    # TypeScript interfaces matching ERP contracts
│   ├── package.json
│   └── tsconfig.json
│
├── docs/                             # Full architectural & engineering specifications
│   ├── ARCHITECTURE.md
│   ├── ERP_REQUIREMENTS.md
│   ├── AI_ERP_INTEGRATION.md
│   ├── DATABASE_DESIGN.md
│   ├── API_DESIGN.md
│   ├── SECURITY.md
│   └── DEVELOPMENT_ROADMAP.md
│
├── docker/                           # Container definitions
│   ├── Dockerfile.erp
│   ├── Dockerfile.ai
│   ├── Dockerfile.frontend
│   └── nginx/
│       └── default.conf
│
├── docker-compose.yml                # Local & staging multi-service orchestration
├── docker-compose.prod.yml           # Production multi-node compose
├── .env.example                      # Complete documented environment template
├── .gitignore                        # Strict rules excluding venv, caches, weights, media
├── README.md                         # Project documentation and quickstart
└── pyproject.toml                    # Unified dependency specification
```

---

### 6. Repository Migration Plan (Phase 0 Findings)

- **Existing State**: The repository is in pristine greenfield state with `SYSTEM_BLUEPRINT.md` staged on branch `main` and no legacy codebase, orphan files, or outdated dependencies.
- **Migration Strategy**: 
  - Standardize directory layout directly according to the production monorepo structure outlined above.
  - Implement zero-dependency drift by establishing shared contracts and schema definitions in `docs/` before implementing code.
  - Ensure development can proceed locally without blocking on unavailable optional hardware (e.g., support CPU fallback for PyTorch/ONNX if NVIDIA CUDA/TensorRT is not present).
