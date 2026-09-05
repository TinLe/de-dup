# Component Catalog

This document details the primary directory structure and component packages of **de-dup**, explaining the roles, interfaces, and public contracts of each layer.

---

## 1. Domain Models (`core/domain/`)

Defines decoupled, immutable, or typed data transfer objects (DTOs) representing entities across the pipeline:

* **`FileDTO`**: Represents a physical file discovered on disk or cached in SQLite.
  - `path: str`: Canonical absolute filesystem path.
  - `size: int`: Byte size of the file.
  - `mtime_ns: int`: Modification timestamp in nanoseconds for cache invalidation.
  - `digest_partial: Optional[bytes]`: 16 KB partial content hash.
  - `digest: Optional[bytes]`: Full content MD5/xxHash digest.
  - `is_ref: bool`: Flag designating reference/pivot files.
* **`DuplicateGroupDTO`**: Encapsulates a duplicate group matching identical contents.
  - `group_id: int`: Unique group identifier.
  - `pivot: FileDTO`: Original reference file (kept unmarked by default).
  - `duplicates: List[FileDTO]`: List of identical matching files (marked for deletion).
  - `saved_bytes: int`: Total potential reclaimable disk space.
  - `serialize_to_dict(group_index: int) -> Dict[str, Any]`: Formats the group for web/API responses.
* **`ScanTaskDTO`**: Complete state and metadata of a scan database task.
  - `task_id: str`, `name: str`, `db_path: str`, `directories: List[str]`.
  - `status: TaskStatus`: One of `IDLE`, `DISCOVERING`, `HASHING`, `SCANNING`, `COMPLETED`, `CANCELLED`, or `FAILED`.
  - `file_count`, `hashed_count`, `match_count`, `dupe_count`: Execution counters.
  - `progress_percentage`, `progress_message`: Live progress tracking.

---

## 2. Storage Engines (`core/storage/`)

* **`DBEngine` (`core/storage/db_engine.py`)**:
  - Connection-isolated SQLite manager with thread-local storage (`threading.local`).
  - Automatically initializes schema, WAL journal mode (`PRAGMA journal_mode=WAL;`), and 60-second busy timeouts.
  - Provides transactional context managers (`with db_engine.transaction() as conn:`).
  - Manages pagination for file lists (`get_files_page`) and duplicate groups (`get_duplicate_groups_page`).
* **`TaskRepository` (`core/storage/task_repo.py`)**:
  - Discovers, registers, and manages SQLite `.db` scan tasks in the user data directory (`~/.local/share/de-dup/scans`).
  - Supports task metadata extraction, task creation, safe deletion, and renaming.
* **`DBVerifier` (`core/storage/db_verifier.py`)**:
  - Validates SQLite schema integrity and foreign key constraints across scan databases.

---

## 3. Pipeline Stages (`core/pipeline/`)

* **`FileDiscovery` (`core/pipeline/discovery.py`)**:
  - Crawls configured target directories or loads cached file metadata from SQLite.
  - Detects new, modified (by `mtime_ns` and `size`), and deleted files during re-scans (`rescan_directories`).
  - Batches database inserts (1,000 files/transaction) to minimize transaction overhead.
* **`ContentHasher` (`core/pipeline/hasher.py`)**:
  - ThreadPool-driven parallel hasher computing two-stage digests (`calc_partial_hash` and `calc_full_hash`).
  - Checks cancellation stop signals frequently to allow immediate user cancellation.
* **`DuplicateMatcher` (`core/pipeline/matcher.py`)**:
  - Queries candidate duplicates using index-accelerated CTE queries.
  - Groups identical files by size and digest, creating `DuplicateGroupDTO` sets.
  - Automatically persists detected groups to the database engine.
* **`CrossDBMatcher` (`core/pipeline/cross_matcher.py`)**:
  - Connects to multiple independent scan databases without modifying them.
  - Detects identical files spanning different directories or historical scans.

---

## 4. Service Layer (`core/service/`)

* **`TaskRunner` & `TaskExecution` (`core/service/task_runner.py`)**:
  - Manages asynchronous scan task threads and execution lifecycles.
  - Isolates execution states (`start_task`, `rescan_task`, `cancel_task`).
  - Caches file and hashed counters in memory during scans to eliminate database lock contention on status polling.
* **`FileDeletionService` (`core/service/deletion.py`)**:
  - Handles safe physical file deletion with live percentage feedback callbacks.
  - Cleans up corresponding database records across attached `DBEngine` instances.

---

## 5. Web & Desktop Interfaces

* **`web/server.py`**:
  - Multi-threaded HTTP server (`ThreadedHTTPServer`) serving static web assets and REST APIs.
  - Thread-safe state synchronization via `ServerState`.
* **`web/static/`**:
  - Single-page application (`app.js`), responsive CSS, and icons.
* **`run_desktop.py`**:
  - Lightweight desktop shell running `pywebview`.
  - Exposes `DesktopBridgeAPI` to JavaScript for native OS folder pickers and system file manager reveals (`reveal_path`).
