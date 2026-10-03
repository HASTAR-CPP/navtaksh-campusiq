# Codebase Inspection & AI Asset Audit Report (Phase 0)
## AI-Powered School Safety, Attendance & Campus Intelligence Platform

---

### 1. Executive Summary & Discovery Findings

Following the extraction and inspection of the existing computer vision research archive (`Object-Detection-Models-main`), a comprehensive audit was executed across all subdirectories, weights, scripts, notebooks, and utility modules.

Key discovery findings:
- **Total Files Inspected**: 66 items (including notebooks, scripts, logs, and model weights).
- **Core Reusable Algorithms**: Production-grade face alignment (`FaceAligner`), FAISS HNSW gallery indexing (`FAISSBuilder`), person-level search & score aggregation (`FaceSearcher`), AdaFace PyTorch architectures, and YOLO+ByteTrack+InsightFace tracking integration (`realtime_log.py`).
- **Binary Assets & Weights**: Discovered active YOLO26 Nano weights (`Servlance/yolo26n.pt`, 5.54 MB).
- **Empty / Incomplete Files**: Identified that `modules/*.py` (10 files) are currently empty stubs (0 bytes) and need proper implementation.
- **Experimental & Duplicate Scripts**: Identified duplicate iterations of live recognition (`live_recog.py` vs `live_recognition.py`, `realtime_face.py` vs `realtime_face_1.py`, `read_log.py` vs `read_log1.py`).
- **Generated Artifacts**: Found untracked `__pycache__` directories in root and `SCRFD/`.

---

### 2. Detailed Asset Categorization & Audit

#### 2.1. Production-Grade Reusable Modules (To Preserve & Migrate)

| File / Component | Source Location | Reusability & Migration Destination |
| :--- | :--- | :--- |
| **`FaceAligner`** | `SCRFD/Phase 1/face_aligner.py` | **High**. Uses `insightface.utils.face_align.norm_crop` for 5-point landmark canonical transformation ($112 \times 112$), face bounding box area verification, and quality categorization (`EXCELLENT`, `GOOD`, `ACCEPTABLE`, `POOR`). Migrate into `ai/ai_service/pipelines/face_pipeline.py`. |
| **`FAISSBuilder`** | `SCRFD/Phase 1/faiss_builder.py` | **High**. Implements vector L2-normalization, cosine similarity duplicate suppression ($\text{threshold}=0.998$), HNSW indexing (`IndexHNSWFlat`, $M=32$, $efConstruction=200$, $efSearch=100$), and gallery serialization. Migrate into `ai/ai_service/models/vector_stores/faiss_store.py`. |
| **`FaceSearcher`** | `SCRFD/Phase 1/face_searcher.py` | **High**. Queries FAISS indices, aggregates top-$k$ matches across multi-sample galleries per person, computes average and peak scores, and filters by confidence margin. Migrate into `ai/ai_service/models/vector_stores/faiss_store.py`. |
| **`AdaFace` Core Architecture** | `SCRFD/Phase 1/adaface.py` | **High**. Pure PyTorch implementation of AdaFace (adaptive margin loss for low-quality video surveillance) with ResNet backbones (`build_model`). Migrate into `ai/ai_service/models/recognizers/adaface.py`. |
| **YOLO + ByteTrack + InsightFace Pipeline** | `Servlance/realtime_log.py` | **High**. End-to-end multi-object tracking using YOLO person detection (class 0), ByteTrack tracking IDs, 10% bounding box contextual expansion, InsightFace `buffalo_l` embedding extraction, and cosine similarity matching against reference galleries. Migrate into `ai/ai_service/pipelines/stream_manager.py`. |
| **YOLO26 Nano Weights** | `Servlance/yolo26n.pt` | **Preserve**. Reusable 5.54 MB PyTorch weights file for ultra-fast edge person detection. Place in `weights/yolo26n.pt`. |
| **ArcFace Verification** | `Servlance/ArcFace.py` | **Moderate**. Straightforward reference implementation comparing 512-d embeddings via dot products of unit vectors. Integrate into recognizer test suite. |
| **Data Cleaning Utilities** | `utils.py` | **Moderate**. Edge-density calculation using Canny edge filtering to prune blank/corrupted frames. Migrate into `ai/ai_service/quality/blur_detector.py`. |
| **SCRFD Pydantic Schemas** | `SCRFD/schemas.py` | **Moderate**. Clean Pydantic schemas for `Point`, `Bbox`, `FaceKeypoints`, and `Face`. Refactor into unified typed schemas in `ai/ai_service/schemas/perception.py`. |

#### 2.2. Duplicate & Experimental Code (To Retain in `experiments/` Archive)

| File / Component | Source Location | Audit Assessment |
| :--- | :--- | :--- |
| `live_recog.py` & `live_recognition.py` | Root | Duplicate scripts testing DeepFace with OpenCV webcam capture and threaded queue. Useful reference for thread-safe frame dropping, but contains hard-coded local paths (`/home/rakesh/...`). Archive into `experiments/deepface_tests/`. |
| `realtime_face.py` & `realtime_face_1.py` | Root | Exploratory scripts comparing SFace and FaceNet. Archive into `experiments/legacy_tests/`. |
| `read_log.py` & `read_log1.py` | `Servlance/` | Log file parsing utilities. Replaced by modern structured JSON logging and ERP audit tables. Archive. |
| `RF-DETR.ipynb`, `DINOv2.ipynb`, `YOLOv26.ipynb`, `YOLOv8.ipynb` | Subdirectories | Exploratory Jupyter research notebooks. Preserve untouched in `experiments/notebooks/`. |
| `modules/*.py` (10 files) | `modules/` | Zero-byte empty files (`config.py`, `face_detector.py`, `faiss_utils.py`, etc.). Replace with real implementations in `ai/ai_service/`. |

---

### 3. Identified Existing Dependencies

Based on inspection of imports across the codebase, the AI service relies on:
- **Core Vision & ML**: `opencv-python`, `numpy`, `torch`, `torchvision`, `torchaudio`, `ultralytics` (YOLOv8 / YOLO26 / ByteTrack), `insightface`, `onnxruntime`, `faiss-cpu` (or `faiss-gpu`), `deepface`.
- **API & Schemas**: `pydantic` (v2), `fastapi`, `uvicorn`, `httpx`, `requests`.
- **System & Utilities**: `pillow`, `scipy`, `matplotlib`.

---

### 4. Code Migration Strategy

```
Object-Detection-Models-main/
        │
        ├── [ Reusable Core CV Logic ]
        │       ├── SCRFD/Phase 1/face_aligner.py  ──► ai/ai_service/pipelines/face_pipeline.py
        │       ├── SCRFD/Phase 1/faiss_builder.py ──► ai/ai_service/models/vector_stores/faiss_store.py
        │       ├── SCRFD/Phase 1/face_searcher.py ──► ai/ai_service/models/vector_stores/faiss_store.py
        │       ├── SCRFD/Phase 1/adaface.py       ──► ai/ai_service/models/recognizers/adaface.py
        │       ├── Servlance/realtime_log.py      ──► ai/ai_service/pipelines/stream_manager.py
        │       ├── Servlance/yolo26n.pt           ──► weights/yolo26n.pt
        │       └── SCRFD/schemas.py               ──► ai/ai_service/schemas/perception.py
        │
        └── [ Research Notebooks & Exploratory Tests ]
                ├── *.ipynb (RF-DETR, DINO, YOLO)  ──► experiments/notebooks/
                └── live_*.py, realtime_*.py       ──► experiments/benchmarks/
```

- **Zero Loss Guarantee**: The original `Object-Detection-Models-main` directory will remain intact during development to preserve historical provenance. Useful components will be extracted into modular, clean, abstract interfaces under `ai/ai_service/` according to the architecture specification.
