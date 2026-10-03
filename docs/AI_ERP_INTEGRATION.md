# AI Vision & ERP Integration Specification
## Contract, Event Protocols, Decision Engine & Biometric Pipelines

---

### 1. Architectural Boundary & Core Principles

```
  +--------------------------------------------------------------------------+
  |                   AI COMPUTER VISION SERVICE (FastAPI)                   |
  |                                                                          |
  |  [ RTSP Capture ] ──► [ YOLO / ByteTrack ] ──► [ SCRFD Face Detection ]  |
  |                                                        │                 |
  |  [ FAISS Vector Match ] ◄── [ ArcFace 512-d ] ◄── [ Quality & Align ]    |
  |            │                                                             |
  |            ▼                                                             |
  |  [ Tracklet-Level Temporal Evidence Accumulator ]                        |
  +--------------------------------------------------------------------------+
                                       │
                                       │ POST /api/v1/ai/attendance-events/
                                       │ (HMAC Signature + Bearer Token)
                                       ▼
  +--------------------------------------------------------------------------+
  |                          SCHOOL ERP (Django + DRF)                       |
  |                                                                          |
  |  1. Event Ingestion & Idempotency Check                                  |
  |  2. Spatial & Timetable Context Resolution:                              |
  |     camera_id ──► Room ──► TimetableSlot(now) ──► Class/Section/Subject  |
  |  3. Attendance Decision Engine:                                          |
  |     - Candidate enrolled in this Class/Section?                          |
  |     - Attendance already recorded for this Period?                       |
  |     - Multi-frame confidence threshold satisfied?                        |
  |  4. State Transition:                                                    |
  |     ├── ACCEPTED        ──► Write AttendanceRecord (PRESENT / LATE)      |
  |     ├── REVIEW_REQUIRED ──► Route to Teacher ReviewQueueItem             |
  |     └── REJECTED        ──► Log Anomaly / Incident                       |
  +--------------------------------------------------------------------------+
```

1. **Non-Invasive Architecture**: The AI Vision service runs as a completely independent process (or container cluster). It possesses **zero database credentials** for PostgreSQL and communicates exclusively via validated HTTP/2 APIs or event streams.
2. **Explicit Separation of Entities**:
   - `RecognitionEvent`: An instantaneous observation of a detected face/person in a single camera frame with bounding box and raw embedding similarity score.
   - `AttendanceEvent`: A consolidated proposal emitted by the AI service when a tracklet exhibits consistent biometric matches over multiple consecutive frames.
   - `AttendanceRecord`: The official academic attendance status committed to the ERP database after timetable resolution and business-rule validation.
3. **No Blind Attribution**: A single frame detection **never** writes an official attendance record. All visual evidence must pass temporal consistency, camera spatial checks, and timetable enrollment validation.

---

### 2. Timetable Context Resolution Protocol

When an `AttendanceEvent` arrives at the ERP, the system must resolve spatial and temporal operational truth:

```
Camera (CAM-102)
   │
   ▼ [Physical Room Mapping]
Room (B-204)
   │
   ▼ [Institutional Master Timetable Query at timestamp T]
TimetableSlot (Period 2: 09:20 - 10:05, Day: Monday)
   ├── Academic Year: 2026-2027
   ├── Class Level: Grade 10
   ├── Section: Section A
   ├── Subject: Mathematics
   └── Assigned Teacher: Dr. Sarah Jenkins
```

#### Context Resolution Algorithm
```python
def resolve_context(camera_id: str, event_time: datetime) -> ResolvedContext:
    # 1. Resolve physical room
    camera = Camera.objects.select_related('room').get(camera_id=camera_id, is_active=True)
    room = camera.room
    if not room:
        return ResolvedContext(status="UNMAPPED_CAMERA", camera=camera)

    # 2. Determine Day of Week and Active Period
    day_of_week = event_time.weekday()  # 0 = Monday ... 6 = Sunday
    time_of_day = event_time.time()

    period = Period.objects.filter(
        start_time__lte=time_of_day,
        end_time__gte=time_of_day,
        is_active=True
    ).first()

    if not period:
        return ResolvedContext(status="NO_ACTIVE_PERIOD", room=room)

    # 3. Query Active Timetable Slot for this Room & Period
    active_year = AcademicYear.objects.filter(is_active=True).first()
    slot = TimetableSlot.objects.select_related(
        'class_level', 'section', 'subject', 'teacher'
    ).filter(
        academic_year=active_year,
        room=room,
        day_of_week=day_of_week,
        period=period
    ).first()

    if not slot:
        return ResolvedContext(status="UNSCHEDULED_ROOM", room=room, period=period)

    return ResolvedContext(
        status="ACTIVE_SESSION",
        room=room,
        period=period,
        class_level=slot.class_level,
        section=slot.section,
        subject=slot.subject,
        teacher=slot.teacher
    )
```

---

### 3. AI Attendance Decision Engine

#### 3.1. Mathematical Thresholds
- $\tau_{\text{accept}}$ (Definitive Match Threshold): $\ge 0.68$ cosine similarity.
- $\tau_{\text{review}}$ (Ambiguous Match Threshold): $[0.45, 0.68)$.
- $\tau_{\text{unknown}}$ (Unknown Rejection Threshold): $< 0.45$.
- Margin threshold $\Delta S = S_1 - S_2 \ge 0.15$ (top match must exceed runner-up match by at least $0.15$ to prevent false identification of similar individuals/siblings).
- Minimum tracklet length: $\ge 15$ frames.
- Consistent match ratio: $\ge 80\%$ of tracked face frames match the same identity candidate.

#### 3.2. State Transition Matrix

| Candidate State | Timetable Context Match | Existing Attendance | Final Decision | System Action |
| :--- | :--- | :--- | :--- | :--- |
| $\text{Similarity} \ge \tau_{\text{accept}}$ & $\Delta S \ge 0.15$ | Student is enrolled in scheduled Class/Section | None recorded | **`ACCEPTED`** | Create `AttendanceRecord` (`PRESENT` or `LATE`). Broadcast to Teacher UI. |
| $\text{Similarity} \ge \tau_{\text{accept}}$ | Student is NOT enrolled in this Class/Section | Any | **`REVIEW_REQUIRED`** | Flag as "Wrong Class Attendance / Roaming". Alert teacher. |
| $\tau_{\text{review}} \le \text{Similarity} < \tau_{\text{accept}}$ | Student is enrolled in scheduled Class/Section | None recorded | **`AMBIGUOUS`** $\to$ **`REVIEW_REQUIRED`** | Insert to Teacher Review Queue with face crop thumbnail. |
| $\Delta S < 0.15$ (Two close candidates) | Either enrolled | None recorded | **`AMBIGUOUS`** $\to$ **`REVIEW_REQUIRED`** | Prompt teacher to select correct identity. |
| Any Similarity | Any | Already marked `PRESENT` | **`REJECTED` (DUPLICATE)** | Discard event idempotently; log audit ping. |
| $\text{Similarity} < \tau_{\text{unknown}}$ | N/A | N/A | **`UNKNOWN`** | Log security observation; do not touch attendance. |

---

### 4. Face Enrollment & Biometric Quality Architecture

#### 4.1. Enrollment Flow
```
Student Intake
   │
   ▼
Capture Multi-Angle Facial Samples (Minimum 3, Maximum 5 photos: Frontal, Left 15°, Right 15°)
   │
   ▼
Face Quality Gatekeeper (Reject sample if ANY check fails):
   ├── 1. Multiple Faces Detected: Reject if count != 1
   ├── 2. Blur Check: Laplacian variance >= 100.0
   ├── 3. Illumination: Mean gray value between 45 and 215, low glare
   ├── 4. Bounding Box Dimensions: Minimum face size >= 120 x 120 px
   ├── 5. Head Pose Estimation: |yaw| <= 20°, |pitch| <= 15°, |roll| <= 15°
   └── 6. Landmark Confidence: 5-point alignment confidence >= 0.90
   │
   ▼
Facial Alignment (5-Point Similarity Transform to 112x112)
   │
   ▼
ArcFace / AdaFace Deep Feature Extractor (512-dimensional float32 vector)
   │
   ▼
Vector L2-Normalization (||v||_2 = 1.0)
   │
   ▼
Store Embedding Metadata in ERP (Model version, embedding version, quality score)
   │
   ▼
Hot-Reload Vector into FAISS Index Gallery (IndexFlatIP / IndexIVFFlat)
```

#### 4.2. Biometric Privacy Guardrails
- **No Direct Embedding Leaks**: Biometric vectors are strictly managed by internal microservice processes and never sent to browser clients or exposed in unauthenticated payloads.
- **Model Versioning**: Every embedding records `model_name` (e.g., `arcface_r100_v1`) and `embedding_version`. If the vision model is updated, the ERP triggers a batch re-indexing workflow.
- **De-enrollment & Right to Erasure**: Deleting a student's biometric profile automatically removes vectors from FAISS index and purges raw enrollment image files.

---

### 5. API Contracts Between AI Service & ERP

#### 5.1. Attendance Event Ingestion
- **Endpoint**: `POST /api/v1/ai/attendance-events/`
- **Headers**:
  - `Authorization: Bearer <AI_SERVICE_INTERNAL_TOKEN>`
  - `X-Signature-SHA256: <HMAC_SHA256_HEX>`
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
  "confidence_margin": 0.430,
  "consecutive_hits": 18,
  "total_track_frames": 22,
  "quality_score": 0.94,
  "face_crop_hash": "a1b2c3d4e5f6789012345678abcdef01",
  "bounding_box": {
    "x_min": 420,
    "y_min": 180,
    "x_max": 560,
    "y_max": 340
  }
}
```
- **Response Body**:
```json
{
  "status": "PROCESSED",
  "decision": "ACCEPTED",
  "attendance_record_id": 10582,
  "student_id": "STU-2026-0042",
  "student_name": "Alexander Hayes",
  "class_name": "Grade 10 - Section A",
  "subject_name": "Mathematics",
  "period": 2,
  "recorded_status": "PRESENT"
}
```

#### 5.2. Recognition Event Ingestion (Perception Logging)
- **Endpoint**: `POST /api/v1/ai/recognition-events/`
- **Request Body**:
```json
{
  "events": [
    {
      "camera_id": "CAM-B204-01",
      "timestamp": "2026-10-05T09:25:29.100Z",
      "track_id": 4821,
      "candidate_id": "STU-2026-0042",
      "similarity": 0.835,
      "box": [418, 179, 558, 338]
    }
  ]
}
```

#### 5.3. Teacher Review Queue Decision
- **Endpoint**: `POST /api/v1/ai/review-queue/{review_id}/resolve/`
- **Headers**: `Authorization: Bearer <TEACHER_JWT>`
- **Request Body**:
```json
{
  "resolution": "APPROVE_AS_STUDENT",
  "student_id": "STU-2026-0042",
  "reason": "Confirmed visually by class teacher - lighting caused lower similarity score."
}
```

---

### 6. Reliability, Fault Tolerance & Edge Buffering

1. **Network Disruption**:
   - The AI vision service maintains a local SQLite ring-buffer for events.
   - If ERP responds with HTTP $5xx$ or connection timeout, events remain queued and retry using exponential backoff with jitter.
2. **Idempotency Guarantee**:
   - Every `AttendanceEvent` contains a unique UUID `event_id`.
   - ERP tracks ingested `event_id` in Redis with a 24-hour TTL and applies PostgreSQL database-level unique constraints on `(student_id, timetable_slot_id, attendance_date)`.
   - Repeated event submissions return the existing decision without creating duplicate attendance rows.
3. **Camera Heartbeat & Watchdog**:
   - AI service sends periodic health pings per camera to `POST /api/v1/cameras/{id}/heartbeat/`.
   - If no heartbeat is received within $45$ seconds, the ERP updates camera status to `OFFLINE` and alerts the Admin and Security consoles.
