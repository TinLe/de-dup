# REST API Reference

The **de-dup** backend runs an asynchronous, multi-threaded HTTP server (`web/server.py`) exposing a clean REST API. This API is consumed by both the glassmorphic HTML Web Console and the native `pywebview` Desktop Shell.

---

## Base URL & Headers

* **Default Base URL**: `http://localhost:8080` (or configured via `--port`)
* **Content-Type**: `application/json`
* **CORS**: All endpoints support cross-origin requests (`Access-Control-Allow-Origin: *`, `OPTIONS` preflight).

---

## Endpoints Overview

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/api/status` | Polling endpoint returning active task state, progress, targets, and system flags. |
| `GET` | `/api/directories` | List currently staged scan target directories. |
| `POST` | `/api/directories` | Add a target directory to the staged list. |
| `POST` | `/api/directories/remove` | Remove a specific target directory by path. |
| `DELETE` | `/api/directories` | Remove a target directory by index, or clear all staged directories. |
| `GET` | `/api/browse` | Interactive file-system folder browser for directory pickers. |
| `GET` | `/api/scans` | List all saved scan task databases and their metadata. |
| `GET` | `/api/scans/<task_id>` | Fetch metadata for a specific scan task database. |
| `POST` | `/api/scans/create` | Create a new scan task database and trigger scanning in the background. |
| `POST` | `/api/scans/rescan` | Re-scan an existing task to detect modified, deleted, or new files. |
| `POST` | `/api/scans/load` | Load a saved scan task database and prepare duplicate groups for viewing. |
| `POST` | `/api/scans/delete` | Delete a saved scan database and its metadata. |
| `GET` | `/api/results` | Paginated duplicate groups for the active or requested task. |
| `GET` | `/api/cache/files` | Paginated file records loaded directly from the scan database. |
| `POST` | `/api/cross_scan` | Run zero-copy duplicate comparison across multiple scan databases. |
| `POST` | `/api/results/delete` | Delete marked duplicate files from disk and update scan database records. |
| `POST` | `/api/scan` | Trigger or resume scanning on the currently active task. |
| `POST` | `/api/scan/cancel` | Request graceful cancellation of the running scan task. |

---

## Detailed Endpoint Documentation

### 1. Status Polling: `GET /api/status`
Returns live state of the active scan task, deletion operations, and staged directory targets. Polled every 500ms by the UI with zero SQL queries during polling.

#### Response Example:
```json
{
  "status": "scanning",
  "scanning": true,
  "deleting": false,
  "cross_matching": false,
  "delete_progress": 0,
  "delete_total": 0,
  "progress": 45,
  "progress_msg": "Comparing candidate duplicates (450/1,000 size groups)...",
  "messages": [],
  "targets": ["/data/photos", "/data/backups"],
  "active_task_id": "photos_backup_2026",
  "has_results": false
}
```

---

### 2. Directory Management

#### `GET /api/directories`
Lists staged target directories.
```json
[
  {"path": "/data/photos", "state": 0},
  {"path": "/data/backups", "state": 0}
]
```

#### `POST /api/directories`
Adds a directory to the staged list.
* **Payload**:
  ```json
  {
    "path": "/data/photos",
    "clear_existing": false
  }
  ```
* **Response**: Updated list of directories.

#### `POST /api/directories/remove`
Removes a directory by exact path.
* **Payload**: `{"path": "/data/photos"}`
* **Response**: Updated list of directories.

#### `DELETE /api/directories?clear_all=true` or `?index=0`
Clears all staged directories or removes a single entry by index.

---

### 3. Filesystem Browser: `GET /api/browse?path=<folder>`
Browses directory trees safely with path traversal protection.

#### Request:
`GET /api/browse?path=/data`

#### Response Example:
```json
{
  "current": "/data",
  "parent": "/",
  "folders": [
    {"name": "backups", "path": "/data/backups"},
    {"name": "photos", "path": "/data/photos"}
  ]
}
```

---

### 4. Scan Task Management

#### `GET /api/scans`
Lists all registered scan databases from the user application directory.
```json
[
  {
    "task_id": "photos_2026",
    "name": "Photos 2026",
    "db_path": "/home/user/.local/share/de-dup/scans/photos_2026.db",
    "directories": ["/data/photos"],
    "status": "completed",
    "file_count": 12500,
    "hashed_count": 12500,
    "match_count": 42,
    "dupe_count": 58,
    "progress_percentage": 100,
    "progress_message": "Scan completed. Found 42 duplicate groups.",
    "db_size_bytes": 1048576,
    "created_at": 1772697600.0,
    "completed_at": 1772697625.0
  }
]
```

#### `POST /api/scans/create`
Creates a new SQLite scan database and launches background scanning.
* **Payload**:
  ```json
  {
    "name": "MyScan",
    "directories": ["/data/photos"],
    "overwrite": false
  }
  ```
* **Response (Success)**:
  ```json
  {
    "success": true,
    "task": {
      "task_id": "MyScan",
      "name": "MyScan",
      "status": "discovering",
      "progress_percentage": 0,
      "directories": ["/data/photos"]
    }
  }
  ```
* **Response (Conflict)**:
  ```json
  {
    "success": false,
    "exists": true,
    "task_name": "MyScan",
    "error": "A scan database named 'MyScan' already exists."
  }
  ```

#### `POST /api/scans/rescan`
Re-scans disk directories against cached database records, automatically detecting new, modified, and deleted files.
* **Payload**: `{"task_id": "MyScan"}`
* **Response**: `{"success": true, "task": { ... }}`

#### `POST /api/scans/load`
Loads an existing scan database into memory without re-scanning.
* **Payload**: `{"task_id": "MyScan"}`
* **Response**: `{"success": true, "is_scanning": false, "task": { ... }}`

#### `POST /api/scans/delete`
Deletes a scan database file and unregisters it.
* **Payload**: `{"task_id": "MyScan"}`
* **Response**: `{"success": true}`

---

### 5. Results & Duplicate Inspection

#### `GET /api/results?limit=50&offset=0&task_id=MyScan`
Returns paginated duplicate groups with reference (pivot) and duplicate file metadata.

#### Response Example:
```json
{
  "success": true,
  "total": 42,
  "total_marked": 58,
  "limit": 50,
  "offset": 0,
  "groups": [
    {
      "id": 0,
      "percentage": 100,
      "files": [
        {
          "path": "/data/photos/IMG_001.JPG",
          "name": "IMG_001.JPG",
          "folder": "/data/photos",
          "size": "4.2 MB",
          "mtime": "",
          "percentage": "100%",
          "is_ref": true,
          "marked": false,
          "markable": false
        },
        {
          "path": "/data/backups/IMG_001_copy.JPG",
          "name": "IMG_001_copy.JPG",
          "folder": "/data/backups",
          "size": "4.2 MB",
          "mtime": "",
          "percentage": "100%",
          "is_ref": false,
          "marked": true,
          "markable": true
        }
      ]
    }
  ]
}
```

#### `GET /api/cache/files?search=.jpg&limit=100&offset=0`
Queries discovered files directly from the active scan database table.

---

### 6. Cross-Database Deduplication: `POST /api/cross_scan`
Runs duplicate comparison across multiple registered scan databases simultaneously without altering original scan files.

* **Payload**:
  ```json
  {
    "db_paths": [
      "/home/user/.local/share/de-dup/scans/scan_a.db",
      "/home/user/.local/share/de-dup/scans/scan_b.db"
    ],
    "limit": 100,
    "offset": 0
  }
  ```
* **Response Example**:
  ```json
  {
    "success": true,
    "total_groups": 15,
    "limit": 100,
    "offset": 0,
    "groups": [ ... ]
  }
  ```

---

### 7. File Deletion: `POST /api/results/delete`
Safely removes marked duplicate files from disk and purges their records from the associated database engines.

* **Payload**:
  ```json
  {
    "task_id": "MyScan",
    "paths": ["/data/backups/IMG_001_copy.JPG"],
    "delete_all_marked": false,
    "db_paths": []
  }
  ```
* **Synchronous Response (<= 50 files)**:
  ```json
  {
    "success": true,
    "deleting": false,
    "deleted_count": 1,
    "task_id": "MyScan"
  }
  ```
* **Asynchronous Response (> 50 files)**:
  ```json
  {
    "success": true,
    "in_background": true,
    "deleting": true,
    "total": 1500,
    "task_id": "MyScan",
    "message": "Deleting 1,500 duplicate files in background..."
  }
  ```
  *(Status and progress can be monitored via `GET /api/status`)*.

---

### 8. Scan Execution Control

#### `POST /api/scan`
Triggers or resumes scanning for the active task.
* **Response**: `{"success": true, "task": { ... }}`

#### `POST /api/scan/cancel`
Signals immediate cancellation to background crawlers and hashers.
* **Response**: `{"success": true}`
