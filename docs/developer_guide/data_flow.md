# Data Flow & Lifecycle Analysis

This document describes the runtime execution lifecycles, background thread model, and data flow pipelines within **de-dup**.

---

## 1. Application Boot Lifecycle

```mermaid
sequenceDiagram
    participant User
    participant Launcher as run_desktop.py / run_web.py
    participant WebServer as web/server.py
    participant State as ServerState
    participant TaskRepo as TaskRepository

    User->>Launcher: Start de-dup (make run / make web)
    Launcher->>WebServer: Initialize ThreadedHTTPServer(port)
    WebServer->>State: Instantiate ServerState (RLock)
    WebServer->>TaskRepo: TaskRepository(scans_dir).refresh()
    TaskRepo-->>WebServer: Loaded existing scan metadata
    WebServer-->>Launcher: Server ready at http://localhost:8080
    Launcher-->>User: Open pywebview Desktop Window or Browser
```

---

## 2. Background Scan Execution Pipeline

When a user stages directories and launches a scan via the web console or desktop shell:

```mermaid
sequenceDiagram
    participant User
    participant REST as POST /api/scans/create
    participant Runner as TaskRunner
    participant Exec as TaskExecution (Worker Thread)
    participant Disc as FileDiscovery
    participant Hash as ContentHasher
    participant Match as DuplicateMatcher
    participant DB as SQLite DBEngine

    User->>REST: POST /api/scans/create {name, directories}
    REST->>Runner: start_task(task_id)
    Runner->>Exec: Spawn background worker thread
    REST-->>User: 200 OK {"success": true, "task": {...}} (immediate)

    Note over Exec,DB: Thread runs asynchronously in background

    Exec->>Disc: collect_files(directories)
    Disc->>DB: Batch insert discovered files
    Disc-->>Exec: Return file list

    Exec->>DB: get_candidate_duplicate_files()
    DB-->>Exec: Return candidates sharing byte size

    Exec->>Hash: hash_candidate_files(candidates)
    Hash->>DB: Compute partial & full digests (batched)
    Hash-->>Exec: Hashes updated

    Exec->>Match: find_duplicates()
    Match->>DB: Query matches via CTE
    Match->>DB: save_duplicate_groups()
    Match-->>Exec: Return DuplicateGroupDTO list

    Exec->>DB: save_task_metadata(status=COMPLETED)
    Exec-->>Runner: Task Completed
```

---

## 3. Real-Time Status Polling Lifecycle

To prevent database lock contention during active scans, polling is completely decoupled from SQLite disk queries:

```mermaid
sequenceDiagram
    participant UI as Web UI / Desktop Shell
    participant REST as GET /api/status
    participant State as ServerState
    participant Exec as TaskExecution

    loop Every 500ms
        UI->>REST: GET /api/status
        REST->>State: Read active_task_id
        REST->>Exec: get_dto()
        Note over Exec: Serves counters directly from in-memory cache<br/>(_hashed_count_cache, progress_percentage)
        Exec-->>REST: ScanTaskDTO snapshot
        REST-->>UI: 200 OK {status, progress, progress_msg, targets}
    end
```

---

## 4. Duplicate Deletion Lifecycle

Deletion operations safely handle single files or large batch operations without blocking the web server:

1. **Request Reception**: `POST /api/results/delete` accepts explicit file paths or `delete_all_marked: true`.
2. **Batch Thresholding**:
   - **Small Batches (<= 50 files)**: Processed synchronously on the HTTP request handler and returned immediately.
   - **Large Batches (> 50 files)**: Dispatched to a background daemon thread (`_run_deletion_pipeline`), returning `in_background: true`.
3. **Execution**: `FileDeletionService` removes files from disk, handles read-only/permission locks, updates progress in `ServerState`, and purges records from associated `DBEngine` instances.
