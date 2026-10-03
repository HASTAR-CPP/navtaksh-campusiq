# Security, Privacy & Compliance Architecture
## AI-Powered School Safety, Attendance & Campus Intelligence Platform

---

### 1. Fundamental Security & Privacy Axioms

1. **Biometric Data Protection (GDPR / FERPA / DPDP Compliance)**:
   Facial recognition data constitutes special-category biometric data. The platform enforces minimal biometric data collection, strict encryption at rest, restricted mathematical vector representations, and automated purge cycles.
2. **Strict Prohibition on Mental-State / Emotion Profiling**:
   **The platform does NOT implement, support, or advertise mental-state, emotion, intent, psychological, or personality inference from surveillance video.** Surveillance and AI vision are strictly restricted to physical presence verification (attendance) and physical safety boundary monitoring (perimeter access control).
3. **Defense in Depth**:
   Layered security across network isolation (VLANs), transport encryption (TLS 1.3), database field encryption (Fernet / AES-256), strict token-based authentication (JWT + HMAC), and immutable audit logging.

---

### 2. Biometric Data Governance & Lifecycle

```
[ Student Enrollment Photo ]
            │
            ▼
[ Quality & Liveness Gate ]
            │
            ▼
[ SCRFD Face Landmark Alignment (112x112) ]
            │
            ▼
[ ArcFace Deep Feature Extraction (512-d float32) ]
            │
            ▼
[ L2-Normalized Mathematical Embedding ] ──► [ FAISS Gallery Index ]
            │                                      │
            ▼                                      ▼
[ Raw Image Purge Policy ]               [ Vector Encryption at Rest ]
(Immediate deletion of raw intake         (AES-256 encryption on FAISS
 photos post-verification unless          gallery index files)
 explicitly retained under parental
 written consent)
```

#### 2.1. Vector Representation vs. Raw Facial Photos
- The operational database **never stores raw facial imagery in relational student tables**.
- The FAISS vector database operates on abstract 512-dimensional floating-point vectors ($v \in \mathbb{R}^{512}, \|v\|_2 = 1.0$). These embeddings are non-invertible mathematical projections.
- Ingestion events pass ephemeral face crop hashes and bounding box coordinates. Temporary review thumbnails are stored in secure, private object storage buckets (MinIO) with pre-signed short-lived URLs ($15$-minute expiration).

#### 2.2. Automated Retention & Purge Cycles
- **Recognition Events**: Raw detection telemetry is purged after $30$ days via automated Celery cron tasks.
- **Review Queue Thumbnails**: Discarded or resolved review thumbnails are purged after $7$ days.
- **Right to Erasure**: Deletion or graduation of a student triggers automated deletion of their `face_enrollments` record, de-indexing of their vectors from FAISS galleries, and physical deletion of any associated reference artifacts.

---

### 3. Network Architecture & Perimeter Isolation

```
 +-------------------------------------------------------------------------+
 |                           CAMERA VLAN (Isolated)                        |
 |  [ RTSP Camera 01 ]   [ RTSP Camera 02 ]   [ Gate Perimeter Camera ]    |
 |  * Zero direct internet route; no public IP assignment.                 |
 +-------------------------------------------------------------------------+
                                      │
                                      ▼ [Internal Switch / Private Subnet]
 +-------------------------------------------------------------------------+
 |                     AI PERCEPTION CLUSTER (Private LAN)                 |
 |  [ FastAPI Inference Nodes ] ──► [ Local Ring Buffers & Hardware Accel] |
 |  * Accessible only by Edge Ingestion & Core ERP.                        |
 +-------------------------------------------------------------------------+
                                      │
                                      ▼ [mTLS / HMAC Internal API]
 +-------------------------------------------------------------------------+
 |                       CORE APPLICATION LAYER (Private)                  |
 |  [ Django Core ERP ] ◄──► [ PostgreSQL 16 ] ◄──► [ Redis 7 & Celery ]   |
 +-------------------------------------------------------------------------+
                                      ▲
                                      │ [Reverse Proxy / TLS 1.3 Termination]
 +-------------------------------------------------------------------------+
 |                       DMZ / PUBLIC GATEWAY (Nginx)                      |
 |  * Rate Limiting (DDoS Protection)                                      |
 |  * WAF Rule Inspection & Security Headers                               |
 +-------------------------------------------------------------------------+
                                      ▲
                                      │ HTTPS (Port 443)
                            [ End-User Clients ]
```

---

### 4. Authentication, Authorization & Secret Management

#### 4.1. Token Security
- **Access Tokens**: Short-lived ($15$ minutes) RS256 or HS256 JWTs carrying user ID, roles, and fine-grained permission codes.
- **Refresh Tokens**: Stored in HTTP-only, secure, `SameSite=Strict` cookies with family rotation to prevent replay attacks.
- **Internal Service Authentication**: Inter-service communication from FastAPI to Django is authenticated via cryptographically signed HMAC headers (`X-Signature-SHA256`) and dedicated service account tokens.

#### 4.2. Camera Credential Encryption
- RTSP passwords are encrypted at rest using AES-256 via Django's cryptographic signing or Fernet symmetric keys.
- Camera credentials are never serialized in public DRF serializers or rendered in browser DOMs.

#### 4.3. Environment & Configuration Hygiene
- **Zero Secrets in Git**: Enforced via pre-commit hooks (`gitleaks` / `.gitignore`).
- Complete template provided in `.env.example` documenting all configuration keys without exposing production secrets.

---

### 5. Tamper-Evident Auditability

Every critical institutional operation logs an immutable record into `audit_logs`:
- **Attendance Overrides**: Must record `user_id`, `student_id`, `timetable_slot_id`, `previous_status`, `new_status`, `override_reason`, `timestamp`, and `client_ip`.
- **Biometric Actions**: Record every enrollment, re-indexing, or revocation.
- **Financial Modifications**: Record all fee waivers, manual invoice adjustments, and payments.
- **Database Permissions**: `audit_logs` table has row-level append-only rules; `UPDATE` and `DELETE` queries on this table are strictly blocked at the database role level.
