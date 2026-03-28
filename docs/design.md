# IoT Spectator System — Design Document

## Overview

IoT Spectator is an intelligent image or video data collector system built for IoT device, like Raspberry Pi. It continuously monitors a connected webcam, captures images or video when motion is detected, and allows users to search captured media using natural language or structured queries — for example, *"when did Amazon deliver my package?"* or *"did anyone pass by around 3 PM?"*

---

## Goals and Constraints

**Goals**
- Run fully self-contained on a single Raspberry Pi with no cloud dependency
- Capture media only when motion is detected (efficient use of storage and compute)
- Support natural language queries on the user's laptop over a local network
- Be extensible: new storage backends, metadata backends, and AI models should be addable without touching core logic

**Constraints**
- An edge device may not have public internet access, but may have local (closed) network connection
- The system must work without any AI: structured queries (by time, labels) always work; AI features are an optional enhancement
- Designed for Python 3.13+

---

## System Architecture

```
┌─────────────────────────────── Raspberry Pi ────────────────────────────────┐
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                  spectator-data-collector                           │   │
│   │                                                                     │   │
│   │   Camera thread ──► asyncio.Queue ──► CapturePipeline (async)       │   │
│   │   (motion detect)                         │                         │   │
│   │                                           ▼                         │   │
│   │                                     spectator-db  ◄── iot-health    │   │
│   │                                           │                         │   │
│   │                               SpectatorService                      │   │
│   │                             ┌──────┴──────┐                         │   │
│   │                           REST            MCP                       │   │
│   └─────────────────────────────┬──────────────┬────────────────────────┘   │
│                                 │              │                            │
└────────────────────────────── LAN ──────────────────────────────────────────┘
                                  │              │
                         ┌────────▼──────────────▼────────┐
                         │         User's Laptop          │
                         │                                │
                         │   REST client / Claude (MCP)   │
                         └────────────────────────────────┘
```

**Key point:** Claude runs on the user's laptop, not on the Pi. It acts as the natural language query engine by calling the Pi's MCP tools over the LAN.

---

## Deployment Model

- Each Pi is **fully self-contained**: one `spectator-data-collector` process, one `spectator-db` instance per device.
- No central server. A future "central brain" to aggregate multiple edge devices is out of scope for V1.
- `spectator-db` and `iot-health` are **libraries** consumed internally by the collector — they are not separate services or processes.

---

## Components

### iot-health

A library for reading device statistics: CPU usage, memory, disk capacity, temperature, and connected cameras. Used by the collector's `SpectatorService` to serve the `/health` endpoint.

---

### spectator-db

An independent, standalone library. Its single responsibility is storing and retrieving media files with their metadata. It has no opinion on how media was captured or what is in it.

> *Unix principle: do one thing, do it well.*

#### Data Model — `MediaRecord`

| Field | Type | Description |
|---|---|---|
| `id` | `str` | UUID, auto-generated on insert |
| `media_type` | `IMAGE \| VIDEO` | Type of media |
| `captured_at` | `datetime` | When captured — provided by the caller |
| `inserted_at` | `datetime` | When stored — set automatically |
| `duration` | `float \| None` | Duration in seconds; video only |
| `format` | `str` | File extension, e.g. `jpg`, `avi` — derived from file |
| `size` | `int` | File size in bytes — derived from file |
| `device_id` | `str \| None` | Which device captured it |
| `labels` | `list[str]` | Semantic labels, e.g. `["person", "car"]` — caller provides |
| `description` | `str \| None` | Text summary — caller provides |
| `embedding` | `list[float] \| None` | Vector embedding for semantic search — caller provides |

**spectator-db never generates labels, descriptions, or embeddings.** It only stores and indexes them.

#### API Surface

```python
# Write
db.insert(file, media_type, captured_at, *, duration, device_id, labels, description, embedding) -> str
db.delete(id) -> None
db.retrieve(id, dest) -> None   # copies file to destination path

# Read
db.get(id) -> MediaRecord
db.query(*, start, end, media_type, device_id, labels, limit, offset) -> list[MediaRecord]
db.search_similar(embedding, *, limit, threshold) -> list[MediaRecord]
```

- `labels` in `query()` uses **ANY-match** semantics (at least one label matches)
- `search_similar` is separate from `query`; hybrid search is a future enhancement
- All `query` parameters are optional and composable

#### Backend Abstractions

Two ABCs decouple the interface from the implementation:

```
Storage (ABC)                   MetadataStore (ABC)
  save(file, mode, name)          insert(record) -> str
  retrieve(name, dest)            get(id) -> MediaRecord
  delete(name)                    delete(id) -> None
  list_all() -> list[Path]        query(...) -> list[MediaRecord]
                                  search_similar(...) -> list[MediaRecord]
```

**V1 implementations:**
- `LocalStorage` — filesystem-backed file storage
- `SQLiteMetadataStore` — SQLite for structured queries; `sqlite-vec` (optional dep) for vector similarity search

**Future:** `S3Storage`, `PostgresMetadataStore` — just new implementations of the same interfaces, no core changes required.

**Rationale for the split:** File storage and metadata storage evolve independently. A Pi deployment uses local filesystem + SQLite. A future cloud deployment might use S3 + Postgres. The `SpectatorDB` facade is unchanged either way.

#### Construction

```python
storage = LocalStorage(path=Path("./media"))
metadata_store = SQLiteMetadataStore(db_path=Path("./spectator.db"))
db = SpectatorDB(storage, metadata_store)
```

---

### spectator-data-collector

The main server process running on each Pi. It owns the full capture lifecycle and exposes a query interface to the outside world.

#### Package Structure

```
collector/
│
├── config.py          # CollectorConfig, CaptureConfig, MotionConfig (+ TOML loader)
├── collector.py       # SpectatorDataCollector — top-level, wires everything
│
├── camera/
│   ├── monitor.py     # CameraMonitor — background thread, motion detection,
│   │                  #   enqueues CaptureTask on asyncio.Queue
│   └── motion.py      # MotionDetector — OpenCV MOG2 background subtraction
│
├── capture/
│   ├── task.py        # CaptureTask dataclass
│   ├── pipeline.py    # CapturePipeline — async queue consumer
│   └── recorder.py    # ImageRecorder, VideoRecorder
│
├── enrichment/
│   ├── base.py        # Enricher ABC + EnrichmentResult
│   └── local.py       # LocalEnricher (on-Pi model — stub, future)
│
├── service.py         # SpectatorService — shared business logic
├── rest.py            # FastAPI app + all routes
└── mcp.py             # MCP server (future)
```

#### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                      single process                         │
│                                                             │
│  ┌──────────────────┐        asyncio event loop             │
│  │  Camera Thread   │    ┌──────────────────────────────┐   │
│  │                  │    │                              │   │
│  │ cv2.VideoCapture │──► │  asyncio.Queue               │   │
│  │ MotionDetector   │    │       │                      │   │
│  │ (blocking I/O)   │    │       ▼                      │   │
│  └──────────────────┘    │  CapturePipeline (coroutine) │   │
│  thread-safe put via     │  REST server (FastAPI)       │   │
│  run_coroutine_          │                              │   │
│  threadsafe()            └──────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

- The **camera thread** runs a blocking OpenCV frame-read loop. When motion is detected, it puts a `CaptureTask` on the async queue using `asyncio.run_coroutine_threadsafe()`.
- The **capture pipeline** runs as an asyncio task. It pulls tasks from the queue and processes them in a thread executor (to avoid blocking the event loop with file I/O).
- The **REST server** (uvicorn/FastAPI) runs in the same event loop. The FastAPI lifespan hook starts the camera thread and pipeline task, and tears them down on shutdown.

#### Capture Flow

```
Camera thread reads frame
        │
        ▼
MotionDetector.detect(frame)
        │
   motion? ──No──► continue reading
        │
       Yes
        │
        ▼
CameraMonitor builds CaptureTask
  IMAGE mode: [trigger_frame]
  VIDEO mode: collect frames for video_duration seconds
        │
        ▼
asyncio.Queue.put(task)
        │
        ▼
CapturePipeline dequeues task
        │
        ├──► ImageRecorder / VideoRecorder → temp file
        │
        ├──► Enricher.enrich() (optional)
        │       └──► labels, description, embedding
        │
        └──► SpectatorDB.insert() → stored + indexed
```

#### AI Enrichment

Enrichment is **pluggable and disabled by default**.

```python
class Enricher(ABC):
    def enrich(self, file: Path, media_type: MediaType) -> EnrichmentResult:
        ...

@dataclass
class EnrichmentResult:
    labels: list[str]
    description: str | None
    embedding: list[float] | None
```

When enabled, enrichment runs **on-Pi** using a local model (e.g. CLIP for embeddings, a small VLM for descriptions). The `LocalEnricher` is a stub awaiting a chosen model backend.

**Rationale:** The Pi may not have internet during operation. Keeping enrichment on-device ensures the system works air-gapped. The interface is deliberately decoupled from any specific model so the backend can be swapped without touching the pipeline.

#### Service Layer

`SpectatorService` is the single point of business logic shared between the REST and (future) MCP layers:

| Method | Description |
|---|---|
| `query_media(...)` | Delegates to `SpectatorDB.query()` |
| `get_record(id)` | Delegates to `SpectatorDB.get()` |
| `retrieve_file(id)` | Copies media to a temp path for serving |
| `device_status()` | Reads device stats via `iot-health` |
| `capture_now()` | Requests an immediate capture from the camera monitor |

#### REST API

| Method | Path | Description |
|---|---|---|
| `GET` | `/` | Welcome message |
| `GET` | `/health` | Device status (CPU, memory, temp, cameras) |
| `GET` | `/media` | Query media records (filters: `media_type`, `device_id`, `limit`, `offset`) |
| `GET` | `/media/{id}` | Get a single record's metadata |
| `GET` | `/media/{id}/file` | Download the media file |
| `POST` | `/capture` | Trigger an immediate capture |

#### MCP Interface (future)

MCP exposes the same operations as REST **except file download** (binary transfers are awkward in MCP). Claude on the user's laptop connects via MCP to perform natural language queries — for example, translating *"show me everyone who came to the door today"* into a structured `query_media()` call with `labels=["person"]` and an appropriate time range.

**Rationale for separate protocols:** MCP and REST serve different clients (Claude vs. generic HTTP). They share the same `SpectatorService` layer — only the transport adapters differ.

#### Configuration

Configuration is a TOML file with Python dataclass defaults underneath:

```toml
[device]
camera_index = 0
device_id = "pi-01"

[motion]
sensitivity = 0.005   # fraction of changed pixels to trigger (0–1)

[capture]
mode = "image"        # "image" or "video"
video_duration = 10.0 # seconds, for video mode

[storage]
media_dir = "./media"
db_path = "./spectator.db"
tmp_dir = "./tmp"

[server]
host = "0.0.0.0"
port = 13722

[enrichment]
enabled = false
```

**Rationale:** TOML is the deployment interface (human-editable, lives on the Pi). The Python API (`CollectorConfig(...)`) is for programmatic use and testing.

---

## Key Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Deployment topology | Self-contained per Pi | Simplicity; no network dependency for capture |
| Motion detection | OpenCV MOG2 (pure Python) | No external daemon; full control in-process |
| Capture trigger | Motion-only (default) | Saves storage and CPU; on-demand via `capture_now()` |
| Video recording | Fixed duration | Simple and predictable for V1; ongoing-motion recording is future |
| AI query engine | Claude on user's laptop via MCP | Pi compute is limited; laptop may have internet for better models |
| Enrichment | Pluggable, on-Pi, disabled by default | Air-gapped operation; optional enhancement |
| Concurrency | Camera thread + asyncio event loop | Camera I/O is inherently blocking; asyncio fits the server model |
| Storage abstraction | `Storage` + `MetadataStore` ABCs | File storage and metadata evolve independently; easy to swap backends |
| Vector search | `sqlite-vec` (optional dep) | Works offline; optional so basic deployments stay lightweight |

---

## Future Considerations

- **Ongoing-motion video recording** — keep recording while motion continues, with a trailing window after it stops
- **Hybrid search** — combine structured filters with vector similarity in a single `query()` call
- **S3Storage / PostgresMetadataStore** — plug in to existing ABCs for cloud-backed deployments
- **Central brain** — aggregate multiple Pi devices; out of scope for V1
- **MCP server** — `mcp.py` stub planned; shares `SpectatorService` with REST
- **LocalEnricher model backend** — integrate CLIP (embeddings) and a small VLM (descriptions) once target hardware is chosen
