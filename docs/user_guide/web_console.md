# HTML Web Console & Desktop Shell Guide

The **HTML Web Console** in **de-dup** provides a full-featured interface for staging directories, managing isolated scan databases, analyzing duplicate groups, and deleting duplicates.

The interface is available in two execution formats:
1. **Desktop Shell (`run_desktop.py`)**: An embedded desktop window using `pywebview` with native OS file selection dialogs and file manager integration (`reveal in file manager`).
2. **Web Browser (`run_web.py`)**: A standalone web application accessible locally or remotely via standard web browsers.

---

## Key Features

* **Glassmorphic Responsive UI**: Dark-mode interface built with Vanilla CSS and modern responsive design.
* **Duplicate Results Studio**: Dedicated studio modal for previewing duplicate pairs side-by-side, analyzing directory provenance, and granular mark selection.
* **Cross-Database Deduplication**: Run deduplication across any combination of saved scan databases concurrently without modifying or re-scanning original databases.
* **Non-Blocking Background Scanning**: Multi-threaded scanner keeps the UI fully responsive with live progress percentages, status messages, and immediate cancel capabilities.
* **Instant Database Loading**: Cached SQLite databases load saved results in under 5 milliseconds.
* **Paginated Results Viewer**: Server-side pagination loads duplicate groups in manageable chunks (default 50 groups/page) to prevent browser memory spikes.

---

## Launching de-dup

### Option 1: Native Desktop Application (pywebview)

```bash
make run
# or
./env/bin/python run_desktop.py
```

* **Command-Line Arguments**:
  - `--port <PORT>`: Custom port for internal backend server (default: auto-detected free port).
  - `--debug`: Enable WebView inspector / developer tools.

### Option 2: Standalone Web Console (Headless / Server Mode)

```bash
make web
# or
./env/bin/python run_web.py --port 8080
```

The Web Console starts by default on `http://localhost:8080` and opens your default web browser.

---

## Step-by-Step User Workflow

### 1. Staging Scan Target Directories
- Click **"Add Folder"** or browse paths via the interactive folder browser.
- In desktop mode, native OS folder pickers will open. In web mode, use the interactive directory browser modal.
- Multiple directories can be added and removed individually or cleared all at once.

### 2. Creating a Named Scan Task
- Enter a name for the scan database (e.g., `Photos_Backup_2026`).
- Click **"Launch Scan"**.
- The scanner runs in the background. Live progress is displayed with real-time file discovery and candidate hashing counts.

### 3. Reviewing Results in the Studio
- Once scanning finishes, duplicate groups appear in the results table.
- Click **"Open Studio"** on any duplicate group to launch the full-screen Duplicate Results Studio.
- Use one-click smart marking presets:
  - **Mark All Duplicates**: Keeps the reference (pivot) file and marks duplicates.
  - **Unmark All**: Clears all marks in the group.
  - **Mark All Except Newest**: Preserves the most recently modified file.
  - **Mark All Except Oldest**: Preserves the original file by modification time.

### 4. Safe Deletion
- Click **"Delete Marked Files"**.
- For small selections (<= 50 files), deletion finishes immediately.
- For large selections, deletion runs in the background with live progress bars.
- Files are deleted from disk, and their records are immediately updated in the database.

---

## Cross-Database Deduplication

To compare files between different disks, historical archives, or backup folders:

1. Navigate to the **"Saved Scans"** tab.
2. Select checkboxes next to two or more completed scan databases.
3. Click **"Run Cross-Scan"**.
4. de-dup compares duplicate candidates across all selected databases in parallel without re-reading physical disks or altering original scan files.
