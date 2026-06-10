# IoT Spectator — Design

**Status:** Draft for review. Grounded in the code currently on `main` across `spectator-db`, `spectator-data-collector`, and `iot-health`. Items marked **(Decision)** need sign-off before implementation; the rest follow mechanically.

---

## Goals

Provide a simple, self-contained way to capture image/video on commodity IoT hardware (e.g. a Raspberry Pi) and make it queryable by time, by event, and by meaning, with optional AI enrichment.

Guiding principles:

- **Per-device autonomy.** Each device owns its data and works with no cloud, no network, and no AI model present. AI is an optional enhancement, never a dependency.
- **Building block over appliance.** `spectator-db` is positioned as a reusable, dependency-light library. Its reason to exist over "files + SQLite" is the *union* neither vector databases nor object stores provide alone: it stores the media file **and** its structured metadata **and** its embedding behind one interface, offline, with no server to run. This deliberately avoids competing with full NVR appliances and instead serves builders who want an embeddable media-intelligence store.
- **Queryable, not just recorded.** Temporal, event/label, and semantic similarity queries are first-class.

**Non-goals:** a central/multi-device database; a model host (the system stores enrichment, never generates it inside `spectator-db`); a continuous streaming/NVR product; an account or cloud service.

---

## Requirements

System-wide requirements. Component-specific detail lives in the per-component sections below.

**Functional**

- Capture image or short video on motion or on demand, and persist each capture with its metadata.
- Store, per capture: media type, capture time, format, size, optional duration, device id, labels, description, and an embedding (with the model identity that produced it).
- Query by composable filters: time range, media type, device id, labels (any-match), with limit/offset, newest-first.
- Similarity search over embeddings, scoped to a single embedding model.
- Attach or replace enrichment (labels, description, embedding) *after* the capture is stored.
- Expose query and control operations to local apps and to an assistant over the LAN (REST and MCP).
- Report device health.

**Non-functional / constraints**

- Runs on a Raspberry Pi-class device; no GPU required.
- `spectator-db` has zero required runtime dependencies; optional extras may accelerate features (e.g. vector search).
- Offline-first: full functionality with no network and no model.
- A failed or slow enricher must never cause a captured media item to be lost.
- Data owned on disk must survive library upgrades (schema versioning + forward migration).
- All datetimes are timezone-aware UTC at the API boundary.
- A documented concurrency contract for the storage layer.

---

## Architecture

Each device runs one collector and one database, plus the health library. The collector is the active agent; the database is passive local memory; health reports vitals.

![IoT Spectator single-device architecture](architecture.svg)

**Layering and dependency direction.** `spectator-db` depends on nothing else in the org. `spectator-data-collector` depends on both `spectator-db` and `iot-health`. The direction never reverses — the database stays ignorant of cameras, models, networks, and other devices.

```
iot-health  ◄───────────────┐  (vitals)
                             │
spectator-db  ◄─── spectator-data-collector
(library, 0 deps)            (capture · enrich · REST · MCP)
```

**Architectural decisions (confirmed in code).**

- **(Decision, confirmed) `spectator-db` is a pure in-process library**, zero runtime dependencies. The collector imports and calls it directly; there is no DB server or wire protocol.
- **(Decision, confirmed) The control plane lives in the collector**, not the database. `rest.py` (FastAPI) and `mcp.py` (FastMCP) both delegate to a single `SpectatorService`, so REST and MCP can never diverge in behavior.
- **No central database.** Devices are autonomous. Central upload (S3) and device-to-device coordination (mesh, leader election) are future, collector-side, and must never leak into `spectator-db`.

### IoT Health

Role: the device's vitals provider. `SpectatorService` calls it for the `device_status` operation surfaced over REST/MCP, and the collector can use it for self-monitoring. It is standalone and has no knowledge of capture or storage.

### Spectator-DB

Role: the device's memory. It receives captures from the collector's pipeline (`store → enrich`), and answers queries for the service layer (`query / get / retrieve / search_similar`). It exposes two backend abstractions — `Storage` (file bytes) and `MetadataStore` (structured records) — orchestrated by the `SpectatorDB` facade. V1 backends are `LocalStorage` and `SQLiteMetadataStore`. It never captures and never runs a model.

### Spectator-Data-Collector

Role: the device's eyes and hands. A camera/motion thread enqueues capture tasks onto an `asyncio.Queue`; the `CapturePipeline` records each task, optionally enriches it, and stores it in `spectator-db`. The same `SpectatorService` that backs storage queries also triggers on-demand captures and reads device health, and is exposed identically over REST and MCP. Cross-device behavior and central upload are out of scope (see that section).

---

## IoT Health

**Responsibilities.** Report platform, CPU architecture, OS, processors, memory, disk capacity, temperature, and cameras through a single `summary()`. Auto-detect the device class (Raspberry Pi vs. generic Linux/Jetson) behind a `BaseHealth` ABC.

**Status & notes.** This is the most mature repo and works today. It uses an older style (classmethods, `Optional[...]`, 2020-era conventions). Treat modernization as opportunistic and low priority; no changes are required for the rest of the system to proceed.

---

## Spectator-DB

The foundation, and where most open decisions live. Today the structure is in place — `Storage`/`MetadataStore` ABCs, a `MediaRecord` model, and a `SpectatorDB` facade with `insert / get / delete / retrieve / query / search_similar`. The decisions below close the gap between that structure and a stable, genuinely reusable 0.1.

### Data model

```python
@dataclass
class MediaRecord:
    media_type: MediaType            # IMAGE | VIDEO
    captured_at: datetime            # tz-aware UTC; when the event happened
    format: str                      # "jpg", "mp4", ...
    size: int                        # bytes
    id: str = <uuid4>
    inserted_at: datetime | None = None   # tz-aware UTC; set by the store
    duration: float | None = None         # seconds; video only
    device_id: str | None = None
    labels: list[str] = []
    description: str | None = None
    embedding: list[float] | None = None
    embedding_model: str | None = None     # NEW — identity of the vector space
    embedding_dim: int | None = None       # NEW — length; enforced on write
    content_hash: str | None = None        # NEW (optional) — sha256 for dedup
```

`embedding`, `embedding_model`, and `embedding_dim` are set together or all `None`. Similarity search compares only vectors sharing the same `embedding_model`.

### Public API (curated, semver-protected)

```python
from spectatordb import (
    SpectatorDB, MediaRecord, MediaType,
    Storage, LocalStorage, SaveMode,
    MetadataStore, SQLiteMetadataStore,
)

SpectatorDB.insert(file, media_type, captured_at, *, duration=None, device_id=None,
                   labels=None, description=None, embedding=None, embedding_model=None) -> str
SpectatorDB.update_enrichment(id, *, labels=UNSET, description=UNSET,
                              embedding=UNSET, embedding_model=UNSET) -> None   # NEW
SpectatorDB.delete(id) -> None
SpectatorDB.get(id) -> MediaRecord
SpectatorDB.retrieve(id, dest) -> None
SpectatorDB.query(*, start=None, end=None, media_type=None, device_id=None,
                  labels=None, limit=None, offset=None) -> list[MediaRecord]
SpectatorDB.search_similar(embedding, *, model, limit=None, threshold=None) -> list[MediaRecord]
SpectatorDB.reconcile() -> ReconcileReport   # NEW — sweep orphan files / dangling rows
```

### Design decisions

- **(Decision) Semantic search: commit, but stage it.** Today `search_similar` raises `NotImplementedError`, so the library's defining feature is dead. Commit to it as core: implement a pure-Python brute-force cosine search in 0.1 (no dependencies; fine over a single device's own low-thousands of vectors), then add `sqlite-vec` as an optional `[vec]` extra in 0.2 for speed, behind the same API. Deferring it entirely would leave the library indistinguishable from files+SQLite.
- **(Decision) Embeddings carry model identity + dimension.** Add `embedding_model` and `embedding_dim`; enforce that the three embedding fields move together and that all vectors for a model share a dimension; `search_similar` takes a required `model`. This must land before the search implementation, since it shapes the schema.
- **(Decision) Store-first, enrich-later.** `MetadataStore` currently has no update path. Add `update_enrichment(...)` (partial update via an `UNSET` sentinel) so a capture can be stored immediately and enriched afterward. This is what lets the collector stop losing captures when enrichment is slow or fails (see the collector section).
- **Atomic insert/delete + reconcile.** The facade currently does `storage.save()` then `metadata.insert()` with no spanning transaction, so partial failure orphans a file or leaves a dangling row. Define the invariant *"every metadata row has a backing file"*: write the file, then metadata; on metadata failure, delete the file (compensating action). On delete, remove metadata first, then file (tolerate a missing file). Crash-time orphans are swept by `reconcile()`. Optional `content_hash` enables idempotent re-ingest and dedup.
- **Schema versioning + migrations.** Only `CREATE TABLE IF NOT EXISTS` exists today. Adopt `PRAGMA user_version` and a small ordered-migration runner; ship 0.1 as schema version 1 with the new columns present. Never break an existing database on upgrade. Put this in before the first release users can pin to.
- **Curated public API + semver.** `__init__.py` is empty, forcing deep imports like `spectatordb.spectatordb.SpectatorDB`. Export the surface above from `spectatordb/__init__.py`, document it as the supported API, and adopt semantic versioning. (Also: the README's `insert` example is stale — it now requires `captured_at`.)
- **Concurrency contract.** A single `sqlite3.Connection` (`check_same_thread=False`, WAL) is shared across collector executor threads and the REST + MCP servers. Given the low, motion-triggered write rate, keep one connection and serialize writes with a `threading.Lock` (WAL still allows concurrent reads); document the store as "thread-safe for the expected low-write workload, one instance per process." Revisit (connection-per-thread) only if write contention appears.
- **UTC at the boundary.** `inserted_at` is UTC-aware but `captured_at` is stored as-passed (possibly naive), making ISO-string range queries fragile. Normalize `captured_at` to UTC on write and document that the API is UTC.

### Roadmap

- **0.1 — honest foundation:** embedding identity, `update_enrichment`, atomicity + `reconcile`, migrations (v1), curated API + semver, concurrency lock, UTC normalization, and pure-Python brute-force `search_similar`. Publish to PyPI.
- **0.2 — acceleration & dedup:** `sqlite-vec` optional `[vec]` backend behind the same API; `content_hash` + idempotent re-ingest; a second `MetadataStore`/`Storage` implementation (even in-memory) to prove the ABCs aren't SQLite-shaped.

---

## Spectator-Data-Collector

The on-device application that turns a camera into stored, queryable media.

**Responsibilities.**

- Read frames from a camera, detect motion, and enqueue `CaptureTask`s.
- Record image or short video to a temp file via the recorders.
- Optionally enrich (labels, description, embedding) via a pluggable `Enricher` (ABC in `enrichment/base.py`; `LocalEnricher` is currently a stub awaiting a model).
- Persist captures to `spectator-db`.
- Expose `query / get / retrieve / device_status / capture_now` through `SpectatorService`, surfaced identically over REST (`rest.py`) and MCP (`mcp.py`).
- Configure via TOML: device id, camera index, motion sensitivity, capture mode/duration, storage paths, server and MCP ports, enrichment on/off.

**Key change — store-first, enrich-later.** The pipeline currently enriches *inline before insert*, so a slow model blocks every capture, and if `enrich()` throws, the pipeline's `except` block drops the capture entirely and the recorded media is unlinked and lost. Once `spectator-db` gains `update_enrichment` (see that section), change the pipeline to: record → `insert` immediately → enrich asynchronously → `update_enrichment`. Enrichment failure then degrades to "stored without labels," never data loss. Pair this with replacing the `LocalEnricher` stub with a real embedding model (e.g. a small CLIP) so 0.1 similarity search has vectors to search.

**Out of scope (future, collector-side, never in `spectator-db`).** Device mesh / health gossip, leader election, multi-device coordination, S3 / central upload, and authentication on the REST/MCP endpoints.

**Open questions.**

1. Embedding model choice (drives `embedding_dim` and on-device cost): a CLIP variant for embeddings, and/or a small VLM for descriptions?
2. Retention: does a device cap storage by age/size and evict (implying a `prune()` API and a role for `reconcile()`), or grow unbounded?
3. Security boundary: REST/MCP currently bind `0.0.0.0` with open CORS — acceptable on a trusted LAN, but is that the stated assumption, or is auth required?
