# AI-Powered School Safety, Attendance & Campus Intelligence Platform

A production-grade, enterprise School ERP and Computer Vision platform engineered with a strict decoupling between the institutional System of Record (Django + DRF) and the high-performance AI perception service (FastAPI + PyTorch/ONNX/FAISS).

---

## 🏛 Architecture & Engineering Documentation

All comprehensive architectural and technical specifications have been authored and are available in the [`docs/`](./docs) directory:

1. [**System Architecture (`docs/ARCHITECTURE.md`)**](./docs/ARCHITECTURE.md)  
   Complete dual-system decoupled architecture, data flow topology, model abstraction interfaces (`Detector`, `FaceDetector`, `FaceRecognizer`, `Tracker`, `VectorStore`), and target monorepo layout.
2. [**ERP Requirements Specification (`docs/ERP_REQUIREMENTS.md`)**](./docs/ERP_REQUIREMENTS.md)  
   Complete specifications for all 16 ERP modules, granular 8-tier RBAC matrix, and business rules.
3. [**AI-ERP Integration & Protocols (`docs/AI_ERP_INTEGRATION.md`)**](./docs/AI_ERP_INTEGRATION.md)  
   Contract between FastAPI and Django, differentiation of `RecognitionEvent` vs `AttendanceEvent` vs `AttendanceRecord`, timetable context resolution algorithm, face enrollment quality gatekeeper, and edge fault-tolerance strategies.
4. [**Database Design & Schema (`docs/DATABASE_DESIGN.md`)**](./docs/DATABASE_DESIGN.md)  
   Normalized PostgreSQL 16 schema, DDL definitions, constraints, composite indexes, soft-delete strategies, and audit tables.
5. [**RESTful API Specification (`docs/API_DESIGN.md`)**](./docs/API_DESIGN.md)  
   Versioned `/api/v1/` endpoints, JWT authentication, request/response JSON schemas, review queue flows, and error contracts.
6. [**Security, Privacy & Compliance (`docs/SECURITY.md`)**](./docs/SECURITY.md)  
   Biometric data governance (vectors vs raw photos), strict ban on emotion/mental-state surveillance, camera credential encryption, network VLAN isolation, and tamper-evident audit logging.
7. [**Development Roadmap & Phasing (`docs/DEVELOPMENT_ROADMAP.md`)**](./docs/DEVELOPMENT_ROADMAP.md)  
   Detailed 13-phase roadmap from inspection to production hardening with verification milestones.
8. [**Codebase Inspection & AI Asset Audit Report (`docs/CODEBASE_INSPECTION_REPORT.md`)**](./docs/CODEBASE_INSPECTION_REPORT.md)  
   Detailed audit of the existing `Object-Detection-Models-main` repository, reusable algorithms (FAISS builder, SCRFD face alignment, AdaFace, ByteTrack), duplicate scripts, and safe migration mapping.

---

## 🚀 Quick Setup & Configuration

Copy the environment template:
```bash
cp .env.example .env
```
Review and configure `.env` variables before launching services.
