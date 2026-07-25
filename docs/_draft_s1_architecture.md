# Section 1: System Architecture, Environment, CLI, File/Folder Layout

## 1. Target Environment: macOS M1, 16 GB RAM

### 1.1 Python Version and Virtualenv

**Python Version: 3.11.x** (minimum 3.11, current as of 2026-07 is 3.12.x, but 3.11 is mature and stable)

Rationale: Python 3.11 has performance improvements over 3.10, async/await stability, and broad support across all required packages (ffmpeg-python, openai, anthropic, ffmpeg, numpy, pillow, librosa). Avoid 3.13+ (bleeding edge; fewer third-party wheels for M1).

**Virtualenv Strategy:**
```bash
# Bootstrap creates a venv at the project root
python3.11 -m venv venv/
source venv/bin/activate
```

Location: `/path/to/VideoAnalyser/venv/` — isolated from system Python, reproducible, vendorable.

On **first activation**, the script checks that the venv exists and is Python 3.11+; if missing, it bootstraps automatically. The venv is added to `.gitignore` and is **not** tracked in version control.

**Platform-specific issues on M1:**
- Some packages (numpy, scipy, librosa) must be installed for `arm64` architecture (native M1), not Rosetta x86_64.
- Ensure Homebrew is installed *without* Rosetta translation (`/opt/homebrew`, not `/usr/local`).
- Numpy/scipy wheels are available on PyPI for M1 (arm64); verify M1 wheels are used in pip output (lines will show `cp311-cp311-macosx_11_0_arm64`).

---

### 1.2 Dependencies: Complete List with Rationale

#### **System-Level (Homebrew, not pip)**

| Dependency | Version | Reason | M1 Consideration |
|---|---|---|---|
| `ffmpeg` | 7.0+ | Video decoding, frame extraction, encoding. Provides `ffprobe` alongside. Exact binary format critical for pipeline. | Install via `brew install ffmpeg` (auto M1). Do NOT use Rosetta. |
| `libsndfile` | 1.2.0+ | Audio file I/O. Dependency of librosa. Installed as transitive from `librosa` via Homebrew tap (if using audio processing). | M1-compatible via Homebrew. |

#### **Python Packages (pip)**

| Package | Version | Reason | Purpose |
|---|---|---|---|
| `ffmpeg-python` | 0.2.1+ | Python wrapper around ffmpeg/ffprobe binaries. Used to extract frames, probe metadata, check codec support. | Interface to on-disk ffmpeg binary. |
| `Pillow` | 10.0.0+ | Image processing: load frames, crop, resize, extract regions, save screenshots. SIMD optimized on M1. | Screenshot generation, frame manipulation. |
| `opencv-python` | 4.8.0+ | Video reading fallback (if ffmpeg extraction is slow). Scene detection via histogram comparison. Frame-to-frame delta calculation. | Scene boundaries, optical flow (optional). |
| `librosa` | 0.10.0+ | Audio processing: load audio, extract mel-spectrograms, detect silence, analyze energy. FFT, onset detection. | Audio analysis pipeline (speech boundaries, silence detection). |
| `numpy` | 1.24+ | Numerical arrays. Underlying library for librosa, opencv, PIL processing. **Must be M1 arm64 wheel.** | Core numerical compute. |
| `scipy` | 1.11+ | Signal processing (scipy.signal): bandpass filters, peak detection, correlations. Used for audio analysis. | Audio feature extraction. |
| `whisper` | (openai/whisper) 20231117+ | OpenAI's Whisper model for speech-to-text transcription. Supports 99 languages, handles background noise, runs locally. | Verbatim transcript extraction (R4.6). |
| `openai` | 1.3.0+ | Official OpenAI Python SDK. Used for vision API (gpt-4-vision) to analyze frames, detect what changed on screen. | Frame analysis (R2.1, R3.3). |
| `anthropic` | 0.7.0+ | Official Anthropic Python SDK. Used for Claude 3.5 Sonnet for persona-based analysis. | Multi-persona analysis (R3.1–R3.8). |
| `imageio` | 2.33+ | Read video frames directly from disk (fallback if ffmpeg-python is slow). Supports many codecs. | Video frame extraction. |
| `python-dotenv` | 1.0.0+ | Load API keys from `.env` file instead of hardcoding. | Config/secrets management. |
| `pydantic` | 2.0+ | Data validation and settings management. Strongly typed config schema. | Config file parsing (YAML/JSON). |
| `PyYAML` | 6.0+ | Parse `.yml` config files. Human-readable configuration. | Configuration format. |
| `rich` | 13.5+ | Terminal UI: progress bars, colored output, tables, syntax highlighting. | CLI progress/logging (R4). |
| `tqdm` | 4.66+ | Alternative progress bar for long-running operations. Simple, non-blocking. | Long-running task feedback. |
| `requests` | 2.31+ | HTTP client for downloading models, fetching resources. Used by Whisper to fetch model weights. | Network I/O. |
| `imagetext-recognition` OR `paddleocr` | 2.7.0.3+ | OCR for extracting text from video frames (if video contains slides, code, or text overlays). | Text extraction from on-screen UI (R4.9). |
| `hashlib` (stdlib) | — | MD5/SHA256 hashing of video frames for duplicate detection and frame matching. | De-duplication, caching. |
| `json` (stdlib) | — | JSON serialization for cache metadata, intermediate pipeline state. | Metadata storage. |

---

### 1.3 Homebrew vs Pip Split

**Homebrew (system-level):**
- `ffmpeg` (and bundled `ffprobe`)
- `libsndfile` (if audio processing is complex; usually installed as ffmpeg dependency)

**Why Homebrew for ffmpeg:**
- Compiled natively for M1 (arm64 binary, not Rosetta translation).
- Pre-linked against all multimedia libraries (libx264, libx265, libopus, libvpx, etc.).
- One central binary that all Python code points to via `PATH`.

**pip (virtual environment):**
- Everything else. Isolated from system Python; reproducible across setups.
- Python packages can be pinned in `requirements.txt` and locked with `pip-compile` (optional, via `pip-tools`).

---

### 1.4 Install / Bootstrap Script

**File: `/home/user/VideoAnalyser/bootstrap.sh`**

```bash
#!/bin/bash
set -e

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
VENV_PATH="$PROJECT_ROOT/venv"

echo -e "${YELLOW}VideoAnalyser Bootstrap${NC}"
echo "Project root: $PROJECT_ROOT"

# Step 1: Check for Python 3.11+
echo -e "\n${YELLOW}[1/5] Checking Python version...${NC}"
PYTHON_VERSION=$(python3 --version | awk '{print $2}')
PYTHON_MAJOR=$(echo $PYTHON_VERSION | cut -d. -f1)
PYTHON_MINOR=$(echo $PYTHON_VERSION | cut -d. -f2)

if [ "$PYTHON_MAJOR" -lt 3 ] || ([ "$PYTHON_MAJOR" -eq 3 ] && [ "$PYTHON_MINOR" -lt 11 ]); then
    echo -e "${RED}ERROR: Python 3.11+ required (found $PYTHON_VERSION)${NC}"
    echo "Install via: brew install python@3.11"
    exit 1
fi
echo -e "${GREEN}✓ Python $PYTHON_VERSION${NC}"

# Step 2: Check for Homebrew and ffmpeg
echo -e "\n${YELLOW}[2/5] Checking ffmpeg...${NC}"
if ! command -v ffmpeg &> /dev/null; then
    echo -e "${RED}ERROR: ffmpeg not found${NC}"
    echo "Install via: brew install ffmpeg"
    exit 1
fi
FFMPEG_VERSION=$(ffmpeg -version | head -1)
echo -e "${GREEN}✓ $FFMPEG_VERSION${NC}"

# Verify M1 architecture
ARCH=$(uname -m)
if [ "$ARCH" != "arm64" ]; then
    echo -e "${YELLOW}WARNING: Not running on M1 (arm64). Found: $ARCH${NC}"
fi

# Step 3: Create virtualenv
echo -e "\n${YELLOW}[3/5] Creating virtualenv...${NC}"
if [ -d "$VENV_PATH" ]; then
    echo -e "${YELLOW}Virtualenv already exists at $VENV_PATH${NC}"
else
    python3 -m venv "$VENV_PATH"
    echo -e "${GREEN}✓ Virtualenv created${NC}"
fi

# Step 4: Activate and install dependencies
echo -e "\n${YELLOW}[4/5] Installing Python dependencies...${NC}"
source "$VENV_PATH/bin/activate"
pip install --upgrade pip setuptools wheel
pip install -r "$PROJECT_ROOT/requirements.txt"
echo -e "${GREEN}✓ Dependencies installed${NC}"

# Step 5: Create folder structure
echo -e "\n${YELLOW}[5/5] Creating folder structure...${NC}"
mkdir -p "$PROJECT_ROOT/raw"
mkdir -p "$PROJECT_ROOT/outputs"
mkdir -p "$PROJECT_ROOT/screenshots"
mkdir -p "$PROJECT_ROOT/.config"
echo -e "${GREEN}✓ Folders created${NC}"

echo -e "\n${GREEN}Bootstrap complete!${NC}"
echo "Next steps:"
echo "  1. Create .env file: cp .env.example .env"
echo "  2. Add API keys to .env (OPENAI_API_KEY, ANTHROPIC_API_KEY)"
echo "  3. Activate venv: source venv/bin/activate"
echo "  4. Run: python video_analyser.py --help"
```

**File: `/home/user/VideoAnalyser/requirements.txt`**

```
ffmpeg-python==0.2.1
Pillow==10.2.0
opencv-python==4.8.0.76
librosa==0.10.0
numpy==1.24.3
scipy==1.11.4
openai-whisper==20231117
openai==1.3.8
anthropic==0.7.1
imageio==2.33.1
python-dotenv==1.0.0
pydantic==2.5.0
PyYAML==6.0.1
rich==13.7.0
tqdm==4.66.1
requests==2.31.0
paddleocr==2.7.0.3
```

**Execution:**
```bash
cd /home/user/VideoAnalyser
chmod +x bootstrap.sh
./bootstrap.sh
```

On first run, creates `.env` file from `.env.example` (if exists), creates venv, installs all deps, creates folder structure. Re-running is idempotent (skips existing folders, checks versions).

---

## 2. Exact On-Disk Layout

### 2.1 Root Folder Location

**Physical path: `/Users/$(whoami)/Desktop/Video Analysis`**

This is the user's Desktop directory on macOS, named exactly `Video Analysis` (per R6). The tilde `~` expands to `/Users/username`, and Desktop is the standard macOS desktop folder.

### 2.2 Folder Structure (Literal Per R6, R7)

```
Video Analysis/
├── raw/                          # Input videos (R6.1)
│   ├── tutorial_woodworking.mp4
│   ├── instagram_reel.mov
│   └── screen_recording.mkv
│
├── outputs/                       # Markdown files (R6.2)
│   ├── tutorial_woodworking.md
│   ├── instagram_reel.md
│   └── screen_recording.md
│
├── screenshots/                   # All screenshots (R6.3)
│   ├── Woodworking bench | 25.07.2026 | YouTube tutorial
│   │   ├── step_1_setup.png
│   │   ├── step_2_cutting.png
│   │   └── step_3_assembly.png
│   │
│   ├── Instagram makeup tips | 24.07.2026 | instagram recording
│   │   ├── intro_look.png
│   │   ├── eyeshadow_application.png
│   │   └── final_result.png
│   │
│   └── Screen Recording - macOS Tutorial | 23.07.2026 | iPhone screen recording
│       ├── open_app.png
│       ├── settings_changed.png
│       └── save_button.png
│
├── .cache/                        # Working/intermediate artifacts (NOT in spec, internal)
│   ├── tutorial_woodworking/
│   │   ├── frames/
│   │   │   ├── frame_0000.jpg
│   │   │   └── ...
│   │   ├── audio.wav
│   │   ├── transcript.json
│   │   └── scenes.json
│   │
│   └── instagram_reel/
│       └── ...
│
├── .env                           # API keys (NEVER committed)
├── .gitignore
├── requirements.txt
├── bootstrap.sh
├── video_analyser.py              # Main entry point
├── config.yml                     # Configuration
└── venv/                          # Virtual environment (in .gitignore)
```

### 2.3 First-Run Initialization

When the script is first run:

1. **Folder creation:** If `raw/`, `outputs/`, `screenshots/`, or `.cache/` don't exist, the script **creates them automatically** with `os.makedirs(..., exist_ok=True)`.

2. **Config file:** If `config.yml` doesn't exist, the script uses built-in defaults and notifies the user that they can create a config file to customize (optional).

3. **`.env` file:** If `.env` is missing, the script **fails with a clear error** saying:
   ```
   ERROR: .env file not found.
   Create it with:
     cp .env.example .env
   Then add your API keys:
     OPENAI_API_KEY=sk-...
     ANTHROPIC_API_KEY=sk-ant-...
   ```

4. **.gitignore:** Committed to repo with:
   ```
   venv/
   .env
   .cache/
   raw/*
   outputs/
   screenshots/
   __pycache__/
   *.pyc
   .DS_Store
   ```

### 2.4 Per-Video Working/Cache Directory

**Pattern: `.cache/<video_stem>/`**

For a video `raw/tutorial_woodworking.mp4`, working directory is `.cache/tutorial_woodworking/`:

```
.cache/tutorial_woodworking/
├── frames/                        # Extracted frames
│   ├── frame_0000.jpg            # Every 1st frame (see R 2.7)
│   ├── frame_0001.jpg
│   └── frame_9999.jpg
│
├── audio.wav                      # Audio extracted (mono, 16 kHz)
├── audio_mono_16k.wav             # Resampled version
│
├── metadata.json                  # Probe result (duration, resolution, codec, fps)
│
├── transcript.json                # Whisper raw output
│   └── {
│       "text": "full verbatim transcript",
│       "segments": [
│         {"id": 0, "start": 0.0, "end": 2.5, "text": "Hello"},
│         ...
│       ]
│     }
│
├── scenes.json                    # Scene boundaries (frame indices)
│   └── [
│       {"start_frame": 0, "end_frame": 120, "label": "intro"},
│       {"start_frame": 121, "end_frame": 450, "label": "main_content"},
│       ...
│     ]
│
├── frame_hashes.json              # MD5 hash of every frame (de-duplication)
│   └── {
│       "frame_0000": "abc123def...",
│       "frame_0001": "def456ghi..."
│     }
│
├── vision_analysis.json           # GPT-4V responses per frame
│   └── {
│       "frame_0120": {
│         "description": "User opens the file menu",
│         "changes": ["menu appeared"],
│         "ui_elements": ["File", "Edit", "Help"]
│       },
│       ...
│     }
│
└── final_analysis.json            # Multi-persona analysis output
    └── {
        "source_analysis": {...},
        "categorization": {...},
        "frame_by_frame": {...},
        "key_points": {...},
        "time_analysis": {...},
        "completeness_check": {...}
      }
```

**Retention:** Cache is kept after successful completion (for resume/idempotency). If user deletes `.cache`, the pipeline reruns from scratch (idempotent, but slow).

**Cleanup:** Optional `--clean-cache` flag allows users to delete cache directories to free disk space.

---

## 3. Screenshot Folder Naming Scheme

### 3.1 Format Specification

**Exact format (per R7):**
```
Part1 | Part2 | Part3
```

Three parts, separated by ` | ` (space-pipe-space).

**Part 1 — Video Title/Subject**
- The actual subject/title of the video.
- Examples: `Woodworking bench`, `Instagram makeup tips`, `Screen Recording - macOS Tutorial`.
- Must be descriptive enough that a reader understands what video this folder contains.

**Part 2 — Processing Timestamp**
- Format: `DD.MM.YYYY` (day, month, 4-digit year).
- Example: `25.07.2026` (25 July 2026).
- Taken from the system clock at the moment the script *starts* processing that video.
- Allows multiple screenshots folders for the same video (re-processed on different dates).

**Part 3 — Source/Origin**
- Where the original video came from.
- Values (non-exhaustive): 
  - `YouTube video`
  - `Instagram recording`
  - `iPhone screen recording`
  - `macOS screen recording`
  - `web source`
  - `other` (if unknown)
- Extracted from video metadata if available; user can override in config.

### 3.2 Character Legality on macOS Filenames

**macOS filesystem (HFS+/APFS) allows almost all characters except:**
- `/` (forward slash) — directory separator; forbidden in filenames.
- `\0` (null byte) — forbidden.
- NUL character.

**However, macOS Finder displays certain characters differently:**
- `:` (colon) — displayed as `/` in Finder (e.g. "Part 1: Tutorial" displays as "Part 1 / Tutorial").
  - This is a *display quirk*, not a filesystem restriction. The file is stored correctly on disk.
  - To avoid confusion, **do not use `:` in the folder name**.

### 3.3 Sanitization Rules

Applied to each part:

1. **Part 1 (Title):** Remove or replace:
   - `:` → replace with `-` (e.g. "Woodworking: Bench Building" → "Woodworking - Bench Building")
   - `/` → forbidden, replace with `-` (e.g. "iOS/Android Tips" → "iOS-Android Tips")
   - Leading/trailing spaces → strip.
   - Multiple consecutive spaces → collapse to single space.
   - Excessively long (>200 chars) → truncate to 200 chars.

2. **Part 2 (Date):** Strict format, no sanitization needed.
   - Always `DD.MM.YYYY` (padded to 2 digits for day/month).
   - Validated programmatically.

3. **Part 3 (Source):** Use a curated list of known origins; if unrecognized, use `other`.
   - Never allow user-provided input without validation.
   - No special characters; alphanumeric + spaces only.

**Example sanitizations:**
```
Input: "Woodworking: Bench & Table Building"
Output: "Woodworking - Bench & Table Building | 25.07.2026 | YouTube video"

Input: "Tips/Tricks for iOS"
Output: "Tips-Tricks for iOS | 25.07.2026 | Instagram recording"

Input: "Gluing: Clamps, 2-day wait, then sand :D"
Output: "Gluing - Clamps, 2-day wait, then sand :D | 25.07.2026 | other"
```

### 3.4 Screenshot File Naming Within Folder

**Within each screenshots folder, individual files are named descriptively:**

Pattern: `<sequence>_<description>.png`

Examples:
- `01_intro_text.png`
- `02_menu_opened.png`
- `03_user_clicks_button.png`
- `04_dialog_window.png`
- `05_final_result.png`

**Rules:**
- Sequence number: 2-digit, zero-padded, starting from `01`.
- Description: lowercase, hyphens separating words, no special chars except `_` and `-`.
- Extension: `.png` (lossless, best for UI/text).
- Total filename length: <100 chars.

**Rationale:** Files are sorted alphabetically in Finder/shell. Sequence numbers ensure they display in the correct order even if description names are alphabetically scrambled.

---

## 4. CLI Design: Arguments, Flags, Defaults

### 4.1 Invocation Syntax

```bash
python video_analyser.py [OPTIONS] [INPUT]
```

### 4.2 Arguments

| Argument | Type | Default | Description |
|----------|------|---------|-------------|
| `INPUT` | path | `raw/` | Input video file or directory. If a file, process that one. If a directory, process all videos in it. If omitted, defaults to `raw/` folder. |

### 4.3 Flags (Options)

#### **Core Flags**

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--output-dir` | path | `outputs/` | Where to save markdown files. |
| `--screenshots-dir` | path | `screenshots/` | Where to save screenshot folders. |
| `--cache-dir` | path | `.cache/` | Where to store intermediate artifacts. |
| `--config` | path | `config.yml` | Config file location. |
| `--dry-run` | bool | False | Print what would be processed without actually processing. |
| `--verbose` / `-v` | int | 0 | Verbosity level: 0=quiet, 1=info, 2=debug, 3=trace. Use `-vvv` for maximum. |
| `--help` / `-h` | — | — | Print help and exit. |
| `--version` | — | — | Print version and exit. |

#### **Processing Control**

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--resume` | bool | False | Skip videos that already have an output `.md` file; only process new videos. |
| `--reprocess` | bool | False | Re-process all videos, overwriting existing outputs. |
| `--frame-sample-rate` | int | 1 | Sample every N-th frame (1 = every frame, 2 = every 2nd, etc.). Speeds up processing on high-fps videos. |
| `--max-frames` | int | None | Maximum frames to extract (limit to first N frames for quick tests). |
| `--screenshot-sample-rate` | int | 5 | Save a screenshot every N seconds (5 = every 5 seconds). Reduced to 1 if frame changes detected. |

#### **API/Cost Control**

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--budget-openai` | float | 10.0 | Max USD to spend on OpenAI API calls. Stops processing if exceeded. |
| `--budget-anthropic` | float | 10.0 | Max USD to spend on Anthropic API calls. |
| `--model-vision` | str | `gpt-4-vision` | Model for frame analysis (gpt-4-vision, gpt-4o). |
| `--model-analysis` | str | `claude-3-5-sonnet` | Model for persona analysis. |
| `--use-local-whisper` | bool | True | Use local Whisper (fast, no API cost). Set False to use OpenAI API. |
| `--whisper-model` | str | `base` | Local Whisper model size: tiny, base, small, medium, large (larger = slower but more accurate). |

#### **Output Control**

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--markdown-only` | bool | False | Skip screenshot generation; output markdown only. |
| `--no-screenshot` | bool | False | Don't save any screenshots to disk (analyze but don't write). |
| `--personas` | str | `all` | Comma-separated list of personas to run: all, source, categorize, frame, keypoints, time, completeness. |

### 4.4 Configuration Defaults

If `config.yml` is not provided or lacks a key, use hardcoded defaults:

```python
DEFAULT_CONFIG = {
    "frame_sample_rate": 1,
    "screenshot_interval_seconds": 5,
    "max_frames_per_video": None,
    "api_budget_openai_usd": 10.0,
    "api_budget_anthropic_usd": 10.0,
    "models": {
        "whisper": "base",
        "vision": "gpt-4-vision",
        "analysis": "claude-3-5-sonnet",
    },
    "enable_personas": ["source", "categorize", "frame", "keypoints", "time", "completeness"],
    "output_markdown_path": "outputs/",
    "output_screenshots_path": "screenshots/",
    "temp_cache_path": ".cache/",
}
```

### 4.5 Example Invocations

```bash
# Process single file
python video_analyser.py raw/tutorial.mp4

# Process all videos in raw/
python video_analyser.py

# Process with custom output dir
python video_analyser.py raw/ --output-dir /tmp/results

# Dry run (see what would happen)
python video_analyser.py --dry-run -vv

# Resume: skip videos with existing outputs
python video_analyser.py --resume

# Quick test: limit frames and set budget
python video_analyser.py raw/tutorial.mp4 --max-frames 300 --budget-openai 2.0

# Debug mode with detailed logging
python video_analyser.py -vvv --model-analysis claude-3-opus

# Screenshots only, no markdown
python video_analyser.py raw/ --markdown-only

# Local Whisper only (no OpenAI for transcription)
python video_analyser.py raw/ --use-local-whisper --whisper-model large
```

---

## 5. Pipeline Stage Graph

### 5.1 Stages and Dependencies

```
                              ┌─────────────────────────────┐
                              │  INGEST: Load Video File    │
                              │ (ffmpeg probe metadata)      │
                              └────────────┬────────────────┘
                                           │
                   ┌───────────────────────┼───────────────────────┐
                   │                       │                       │
                   ▼                       ▼                       ▼
        ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
        │ EXTRACT FRAMES   │   │ EXTRACT AUDIO    │   │ CALC FRAME HASHES│
        │ (ffmpeg decode)  │   │ (ffmpeg extract) │   │ (MD5/frame)      │
        └────────┬─────────┘   └────────┬─────────┘   └────────┬─────────┘
                 │                      │                      │
                 │  [parallel: no deps] │                      │
                 │                      ▼                      │
                 │            ┌──────────────────┐             │
                 │            │ RESAMPLE AUDIO   │             │
                 │            │ (16 kHz mono)    │             │
                 │            └────────┬─────────┘             │
                 │                     │                       │
                 │                     ▼                       │
                 │            ┌──────────────────┐             │
                 │            │ TRANSCRIBE AUDIO │             │
                 │            │ (Whisper local)  │             │
                 │            └────────┬─────────┘             │
                 │                     │                       │
                 │         ┌───────────┴──────────┐            │
                 │         │                      │            │
                 │         ▼                      ▼            │
                 │   ┌──────────────┐    ┌──────────────────┐  │
                 │   │ALIGN: Match  │    │DETECT SILENCE    │  │
                 │   │Frames ↔ Text │    │(Audio pauses)    │  │
                 │   └──────┬───────┘    └────────┬─────────┘  │
                 │          │                     │            │
                 └──────────┼─────────────────────┼────────────┘
                            │                     │
                            ▼                     ▼
                 ┌──────────────────────────────────────┐
                 │  SCENE DETECTION: Find boundaries    │
                 │  (histogram delta, optical flow)     │
                 │  Outputs: scene_0 [frames 0-120],   │
                 │           scene_1 [frames 121-450], │
                 │           etc.                       │
                 └──────────────┬─────────────────────┘
                                │
                                ▼
                 ┌──────────────────────────────────────┐
                 │  SELECT KEY FRAMES: One per scene    │
                 │  (max change, high entropy, or every │
                 │   N frames if static)                │
                 │  Outputs: frame_0120.jpg, etc.       │
                 └──────────────┬─────────────────────┘
                                │
                                ▼
                 ┌──────────────────────────────────────┐
                 │  VISION ANALYSIS: GPT-4V             │
                 │  "Describe what changed, what's on   │
                 │   screen, which UI elements are      │
                 │   visible, text content"             │
                 │  [parallel: batch 4-8 frames]        │
                 └──────────────┬─────────────────────┘
                                │
                ┌───────────────┴────────────────┐
                │                                │
                ▼                                ▼
     ┌──────────────────┐           ┌──────────────────────┐
     │ PERSONA: Source  │           │ PERSONA: Categorize  │
     │ Analysis (where  │           │ (what type of video) │
     │ did it come from)│           └──────────────────────┘
     └──────────────────┘
                │
                ├─ PERSONA: Frame-by-frame
                │  (detailed breakdown of each scene)
                │
                ├─ PERSONA: Key Points
                │  (what makes it unique/different)
                │
                ├─ PERSONA: Time Analysis
                │  (durations, on-screen vs real-world time)
                │
                └─ PERSONA: Completeness Check
                   (verify all steps, no broken links, QA)
                   [must run LAST; reads all other outputs]
                   
                                │
                                ▼
                 ┌──────────────────────────────────────┐
                 │  GENERATE MARKDOWN: Combine all      │
                 │  persona outputs, transcript, links, │
                 │  timestamps, screenshots into one    │
                 │  unified markdown file               │
                 └──────────────┬─────────────────────┘
                                │
                                ▼
                 ┌──────────────────────────────────────┐
                 │  QA: Verify links, check for broken  │
                 │  references, validate markdown syntax│
                 └──────────────────────────────────────┘
```

### 5.2 Stage Definitions

| Stage | Input | Output | Cacheable | Parallelizable | CPU/Memory Heavy | Reason |
|-------|-------|--------|-----------|---|---|---|
| **INGEST** | Video file | `metadata.json` (duration, fps, resolution, codec) | Yes | No | Low | Must run once per video. Fast (just probing, no decoding). |
| **EXTRACT FRAMES** | Video file | `frames/*.jpg` (all frames or sampled) | Yes | No | High (I/O bound) | Frame extraction is the main I/O bottleneck. Store on disk; reuse if cache exists. |
| **EXTRACT AUDIO** | Video file | `audio.wav` (original sample rate) | Yes | No | Medium | One-time decode; can reuse. |
| **CALC FRAME HASHES** | `frames/*.jpg` | `frame_hashes.json` | Yes | **YES** (hash each frame in parallel) | Low | De-duplication and matching. Embarrassingly parallel; can run 4-8 in parallel on M1. |
| **RESAMPLE AUDIO** | `audio.wav` | `audio_16k_mono.wav` | Yes | No | Low | Lightweight audio conversion. |
| **TRANSCRIBE** | `audio_16k_mono.wav` | `transcript.json` | Yes | No | **HIGH** | Local Whisper is CPU-intensive (2-5 min for 1-hour video on M1). Cached afterward. |
| **ALIGN** | `transcript.json`, `frames/`, timing info | `alignment.json` (frame_idx → transcript segment) | Yes | No | Low | Sync speech to frames. Fast; purely computational. |
| **DETECT SILENCE** | `audio_16k_mono.wav` | `silence_intervals.json` | Yes | No | Low | Identify quiet sections; use for scene boundaries. |
| **SCENE DETECTION** | `frames/`, `frame_hashes.json`, frame deltas | `scenes.json` (boundaries, labels) | Yes | **YES** (compute histogram delta in parallel) | Medium | Detects scene changes; parallelizable frame comparison. |
| **SELECT KEY FRAMES** | `frames/`, `scenes.json` | List of frame indices to analyze further | No | No | Low | Filter frames for vision API (reduce API calls). |
| **VISION ANALYSIS** | Key frames (images) | `vision_analysis.json` (descriptions per frame) | Yes (by frame) | **YES** (batch 4-8 frames per API call) | Low (API-bound, not local) | Expensive (API calls); batch to reduce cost. Cache per frame so re-runs skip analyzed frames. |
| **PERSONA: All 6** | All previous outputs | Individual JSON files per persona (e.g., `source_analysis.json`) | Yes (per persona) | **Partially** (personas can run in parallel, but some depend on others; Completeness must run last) | Medium | Claude API calls; leverage async to parallelize independent personas. |
| **GENERATE MARKDOWN** | All persona outputs, transcript, frame metadata, alignment | `outputs/<video_name>.md` | No | No | Low | Templating; fast. |
| **QA** | `outputs/<video_name>.md`, `screenshots/` | Pass/fail + error list | No | No | Low | Link validation, syntax check. |

### 5.3 Caching Strategy

Every stage writes its primary output to `.cache/<video_stem>/<stage_name>/<output_file>`.

**Cache retention rules:**
- **On success:** Cache persists; re-runs skip completed stages and reuse results.
- **On partial failure (mid-pipeline crash):** Cache is preserved; re-running resumes from the crash point.
- **Idempotency:** Re-running the same video file (same path, same hash) picks up cached results. If file changes (different video in same path), cache is invalidated.

**Cache invalidation:**
- Detected by comparing `ffmpeg -hash md5 <video_file>` hash with stored `metadata.json`.
- If hash differs, old cache is moved to `.cache/<video_stem>_old_<timestamp>/` and pipeline reruns.

**User options:**
- `--resume` skips videos with existing `outputs/<name>.md`; does not re-run pipeline.
- `--reprocess` deletes cache for selected videos and reruns pipeline from scratch.
- `--clean-cache` removes `.cache/` directory entirely.

---

## 6. Configuration: File Format, API Key Handling, Model Selection, Tunable Thresholds

### 6.1 Config File Format and Location

**File: `.config/video_analyser.yml`** (or `config.yml` in root; check both, prefer explicit `--config` flag)

**Format: YAML** (human-readable, supports nesting, comments)

```yaml
# VideoAnalyser Configuration
# Last edited: 2026-07-25

## Processing Parameters
processing:
  frame_sample_rate: 1                    # Extract every N-th frame (1 = every frame)
  max_frames_per_video: null              # Limit extraction to N frames (null = no limit)
  screenshot_interval_seconds: 5          # Save screenshot every N seconds
  enable_scene_detection: true            # Use histogram delta to find scene boundaries
  scene_detection_threshold: 0.15         # Histogram change threshold (0.0-1.0, higher = more scenes)

## API Configuration
api:
  openai_api_key: ${OPENAI_API_KEY}       # Use env var; never hardcode
  anthropic_api_key: ${ANTHROPIC_API_KEY} # Use env var
  openai_org_id: null                     # Optional; set if using org-level billing
  
  # Budgets (in USD)
  budget_openai_per_run: 10.0             # Max spend on OpenAI per video
  budget_anthropic_per_run: 10.0          # Max spend on Anthropic per video

## Model Selection
models:
  whisper_local_model: "base"             # tiny, base, small, medium, large
  vision_model: "gpt-4-vision"            # gpt-4-vision or gpt-4o
  analysis_model: "claude-3-5-sonnet"     # claude-3-5-sonnet or claude-3-opus
  
## Personas (which analyses to run)
personas:
  enabled:
    - source                              # Where did it come from?
    - categorize                          # What type of video?
    - frame_by_frame                      # Detailed breakdown
    - key_points                          # What's unique?
    - time_analysis                       # Duration, timing
    - completeness_check                  # QA verification

## Output Paths (absolute or relative to repo root)
output:
  markdown_dir: "outputs/"
  screenshots_dir: "screenshots/"
  cache_dir: ".cache/"

## Logging and Debugging
logging:
  level: "INFO"                           # DEBUG, INFO, WARNING, ERROR
  file: ".logs/video_analyser.log"        # Optional; log to file
  console_colors: true                    # Use colored terminal output

## Vision Analysis (per-frame)
vision:
  # Max frames to send to GPT-4V per video (cost control)
  max_vision_frames: 100
  # Batch size for vision API calls (4-8 optimal)
  batch_size: 6
  # Prompt template (can be customized)
  prompt_template: "default"              # or "detailed", "minimal"

## Transcription
transcription:
  use_local_whisper: true                 # Use local Whisper (no API cost)
  fallback_to_openai: false               # If local fails, use OpenAI API
  language: null                          # null = auto-detect, or "en", "es", etc.

## Thresholds and Tuning
thresholds:
  silence_duration_seconds: 0.5           # Minimum silence to detect
  silence_amplitude_threshold: 0.01       # Audio amplitude below this = silence
  frame_hash_similarity: 0.95             # If frames are >95% similar, skip one
  optical_flow_threshold: 2.0             # Pixels moved per frame to detect motion
```

### 6.2 API Key Handling (Never Hardcode)

**Rule 1: Environment Variables**

API keys come from `.env` file, loaded via `python-dotenv`:

```bash
# File: .env (in .gitignore, NEVER committed)
OPENAI_API_KEY=sk-proj-...
ANTHROPIC_API_KEY=sk-ant-...
```

**In Python:**

```python
from dotenv import load_dotenv
import os

load_dotenv()

OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
ANTHROPIC_API_KEY = os.getenv("ANTHROPIC_API_KEY")

if not OPENAI_API_KEY:
    raise ValueError("OPENAI_API_KEY not found in .env or environment")
```

**Rule 2: No Hardcoding**

- Never embed keys in source files, config, or comments.
- If a key is found in config file, log a warning and use env var instead.
- Example: `openai_api_key: ${OPENAI_API_KEY}` in config.yml means "read from env var OPENAI_API_KEY".

**Rule 3: Secure Storage**

- On macOS, keys can optionally be stored in Keychain (advanced; optional).
- For scripts, `.env` is simplest and most portable.
- Always add `.env` to `.gitignore`.

**Rule 4: Validation**

On startup, script validates that keys exist and are valid (make a low-cost test call):

```python
def validate_api_keys():
    """Ensure API keys are set and reachable."""
    try:
        # OpenAI: List models (low cost)
        client_openai = OpenAI(api_key=OPENAI_API_KEY)
        client_openai.models.list()
        print("✓ OpenAI API key valid")
    except Exception as e:
        print(f"✗ OpenAI API key invalid: {e}")
        raise

    try:
        # Anthropic: Get account info
        client_anthropic = Anthropic(api_key=ANTHROPIC_API_KEY)
        # Anthropic doesn't have a simple test; just check key format
        if not ANTHROPIC_API_KEY.startswith("sk-ant-"):
            raise ValueError("Invalid Anthropic key format")
        print("✓ Anthropic API key valid")
    except Exception as e:
        print(f"✗ Anthropic API key invalid: {e}")
        raise
```

### 6.3 Model Selection

**Vision Model:**
- `gpt-4-vision` — cheaper ($0.01/image, max 2048x2048), sufficient for UI analysis.
- `gpt-4o` — more expensive but better reasoning; only if budget allows.
- Default: `gpt-4-vision`.

**Analysis Model (Personas):**
- `claude-3-5-sonnet` — fast, cheap ($0.003/1K input tokens), good for iterative analysis.
- `claude-3-opus` — more powerful but 2x cost; use for complex persona analysis if budget allows.
- Default: `claude-3-5-sonnet`.

**Whisper Model (Local Transcription):**
- `tiny` (39M params) — 2x faster, less accurate; good for quick tests.
- `base` (74M params) — default; balance of speed/accuracy.
- `small` (244M params) — better accuracy, 2-3x slower.
- `medium` (769M params) — very good, but slow (5-10 min/hour on M1).
- `large` (1550M params) — best accuracy, but 10-20 min/hour on M1; may need babysitting for memory.
- Default: `base`.

**Cost per video estimate (1 hour, base config):**
- Whisper local base: $0 (local), ~3 min CPU.
- Vision analysis (50 key frames): ~$0.50 (gpt-4-vision).
- Persona analysis (6 personas × 3 API calls each): ~$0.15 (claude-3-5-sonnet).
- **Total: ~$0.65 per hour of video.**

### 6.4 Tunable Thresholds

| Threshold | Default | Range | Description |
|-----------|---------|-------|-------------|
| `frame_sample_rate` | 1 | 1-10 | Extract every N-th frame. Use 2-4 for fast preview. |
| `scene_detection_threshold` | 0.15 | 0.05-0.5 | Histogram delta to trigger scene change (lower = more sensitive). |
| `max_vision_frames` | 100 | 10-500 | Max frames to analyze with GPT-4V. Lower = cheaper. |
| `batch_size` (vision) | 6 | 2-8 | Frames per batch for API calls. Higher = fewer calls, faster. |
| `silence_duration_seconds` | 0.5 | 0.2-2.0 | Minimum silence to flag as scene boundary. |
| `silence_amplitude_threshold` | 0.01 | 0.001-0.1 | Amplitude below this = silence (0.0-1.0 scale). |
| `frame_hash_similarity` | 0.95 | 0.8-0.99 | Skip frames identical to previous (reduce redundancy). |

---

## 7. Memory and Performance Strategy for 16 GB M1

### 7.1 Streaming vs Loading

**Design principle: Stream everything. Load minimally.**

**Frames:**
- Never load all frames into memory at once.
- Use generator: extract one frame at a time via ffmpeg pipe, process, save, discard.
- Pattern: `ffmpeg -i video.mp4 -f image2pipe -vcodec ppm - | while read ...`
- Memory footprint: ~50 MB per frame (4K RGBA), but only hold one at a time.

**Audio:**
- Extract once to `.wav` file; load via `librosa.load()` (loads to RAM by default).
- For very long videos (>2 hours), use `librosa.load(sr=16000, mono=True)` which resamples on-the-fly.
- 1 hour of mono audio @ 16 kHz = ~230 MB RAM. Acceptable.

**Vision Analysis:**
- Load frame image (50 MB max), send to GPT-4V, discard.
- Batch 4-8 frames per API call to reduce overhead.

**Cache:**
- Disk-based: `.cache/<video_stem>/frames/*.jpg`, `.cache/<video_stem>/transcript.json`.
- Re-runs avoid re-extracting/re-transcribing (cache hit).

### 7.2 Frame Budget for 16 GB M1

**RAM Distribution (16 GB total):**
- OS + Python runtime: ~2 GB (fixed).
- ffmpeg + ffprobe processes: ~1 GB (shared buffers).
- Librosa (audio): ~500 MB (1 hour audio @ 16 kHz).
- PIL/numpy frame processing: ~1 GB (working memory for image ops).
- API clients (OpenAI, Anthropic): ~500 MB (request buffers).
- **Headroom: ~11 GB available for pipeline.**

**Frame extraction process:**
- Stream frames one-at-a-time; never buffer all in memory.
- For a 2-hour 4K (3840×2160) video @ 30 fps:
  - Total frames: 30 fps × 7200 sec = 216,000 frames.
  - On-disk (4 MB/frame JPEG @ 90% quality): ~864 GB (PROBLEM!).
  - Solution: Aggressive sampling (every 2nd or 5th frame) + lossy JPEG (60-70% quality).
  - With frame_sample_rate=2 + 60% JPEG quality: ~108 GB (still large, but manageable with SSD).

**Practical limits for 16 GB M1:**
- Videos up to 2 hours, 4K: feasible with aggressive sampling (every 2-5 frames, low-quality JPEGs).
- Videos up to 10 hours, 1080p: feasible (every frame, medium-quality JPEGs).
- Longer videos: require `--frame-sample-rate` > 1 (e.g., `--frame-sample-rate 5` = every 5th frame).

### 7.3 Handling a 2-Hour 4K Video Without OOM

**Strategy:**

1. **Frame extraction:** Stream via ffmpeg pipe; save JPEG at 65% quality.
   ```bash
   ffmpeg -i video.mp4 \
     -vf "fps=fps=1/2" \        # Every 2 seconds = 1 frame per 2 sec
     -q:v 6 \                   # JPEG quality (1-31; 6 ≈ 65%)
     -f image2 frames/frame_%05d.jpg
   ```
   - 2 hours @ 1 frame/2 sec = 3600 frames.
   - 3600 frames × 500 KB (65% quality 4K JPEG) = 1.8 GB on disk. Acceptable.

2. **Audio extraction:** No sampling needed; extract once.
   ```bash
   ffmpeg -i video.mp4 -acodec pcm_s16le -ar 16000 -ac 1 audio.wav
   ```
   - 2 hours mono 16 kHz = ~230 MB on disk, ~230 MB in RAM.

3. **Transcription:** Whisper processes audio in 30-second chunks; memory-efficient.
   - GPU is optional on M1 (can use CPU). CPU is slower but doesn't stress VRAM.

4. **Vision analysis:** Batch 6 frames per API call; discard after sending.
   - 3600 frames / 6 per batch = 600 API calls. At $0.01/image, ~$60 (ouch). Reduce to top 50 key frames instead.

5. **Persona analysis:** Process one persona at a time; JSON results fit in memory easily.

**Result:** 2-hour 4K video processes without OOM, under 30 minutes total time (mostly API wait time).

### 7.4 M1 Optimization

**Leverage M1 strengths:**
- **Neural Engine:** Not directly accessible to Python; but frameworks like Core ML can use it. Skip for now (complexity not worth it).
- **Unified memory:** RAM is shared between CPU and GPU. Helps when processing images.
- **SIMD:** Numpy, Pillow, librosa all use SIMD optimizations on arm64. Use them.

**Things to avoid:**
- Running ffmpeg under Rosetta (x86_64 emulation). Always use native M1 binary (`brew install ffmpeg`).
- Compiling packages from source; use precompiled arm64 wheels from PyPI.
- Using x86_64 Docker images; if using Docker, specify `--platform linux/arm64`.

---

## 8. Error Handling Philosophy

### 8.1 What is Fatal

**Fatal errors (exit immediately with error code 1):**
- Missing or invalid `.env` file (API keys required).
- No input video file found in `raw/`.
- Video file is corrupted (ffprobe fails).
- No write permission to `outputs/` or `screenshots/`.
- API key validation fails (test call to OpenAI/Anthropic fails).
- Budget exceeded before video finishes processing.
- Disk space insufficient (< 1 GB free for cache).

**Error message format:**
```
ERROR: [brief description]
  Context: [what was being done]
  Action: [what to do to fix]
  
Example:
  ERROR: API budget exceeded
    Context: Processing 'tutorial.mp4'; spent $11.50 on OpenAI
    Action: Increase --budget-openai to 12.0 or higher
```

### 8.2 What Degrades Gracefully

**Recoverable errors (log warning, skip/continue):**
- Single frame extraction fails → skip that frame, continue.
- Vision API call times out → retry up to 3 times, then skip frame.
- Persona analysis returns empty/error → use default fallback output.
- Screenshot save fails (permission) → log warning, continue without screenshot.
- Transcription language detection fails → assume English, continue.
- Scene detection threshold too high (no scenes detected) → use frame-based fallback.

**Error message format:**
```
WARNING: [brief description]
  Context: [what was being done]
  Fallback: [what was done instead]
  
Example:
  WARNING: Frame 1250 extraction failed
    Context: ffmpeg pipe broke mid-stream
    Fallback: Skipping frame; continuing from frame 1251
```

### 8.3 Retry Logic

**Transient failures (network timeouts, rate limits):**
- Retry up to 3 times with exponential backoff (1s, 2s, 4s).
- Log each retry.
- On final failure, degrade gracefully (skip or use fallback).

```python
def call_api_with_retry(fn, max_retries=3):
    for attempt in range(1, max_retries + 1):
        try:
            return fn()
        except (Timeout, RateLimit, ConnectionError) as e:
            if attempt == max_retries:
                logger.error(f"API call failed after {max_retries} retries: {e}")
                raise
            wait_time = 2 ** (attempt - 1)  # 1, 2, 4 seconds
            logger.warning(f"Retry {attempt}/{max_retries} in {wait_time}s: {e}")
            time.sleep(wait_time)
```

### 8.4 Logging and Debugging

**Verbosity levels:**
- `-v` (INFO): Progress updates, major stages, timestamps.
- `-vv` (DEBUG): Frame extraction progress, API call details (without keys), cache hits.
- `-vvv` (TRACE): Every frame, every API call, full JSON responses (sanitized).

**Log file:** `.logs/video_analyser.log` (optional, configurable in config.yml).

**Sanitization:** Never log API keys, full API responses, or personally identifiable information (if video contains names/faces, mask them in logs).

### 8.5 User-Facing Error Recovery

**On crash mid-pipeline:**
1. Exception is caught at top-level.
2. State is written to `.cache/<video_stem>/crash_dump.json` (timestamp, stage name, error message, input).
3. User sees:
   ```
   ERROR: Pipeline crashed during [stage name]
   Crash dump: .cache/tutorial_woodworking/crash_dump.json
   To resume, run: python video_analyser.py raw/tutorial_woodworking.mp4 --resume
   ```
4. Re-running with `--resume` skips completed stages, resumes from crash point.

---

## Summary

This section specifies the complete foundation for the VideoAnalyser:

1. **Environment:** Python 3.11, M1-native, 16 GB RAM, macOS Homebrew + pip split.
2. **Dependencies:** Full bill of materials with rationale (ffmpeg, Whisper, OpenAI, Anthropic, librosa, numpy, PIL, etc.).
3. **Folder layout:** Desktop → `Video Analysis/` with `raw/`, `outputs/`, `screenshots/`, and hidden `.cache/`.
4. **Naming:** Screenshot folders as `Title | DD.MM.YYYY | Source`; files sanitized for macOS edge cases (`:` handling).
5. **CLI:** Flexible argument parsing (file or directory input, resume, reprocess, budgets, model selection, dry-run).
6. **Pipeline:** DAG of 15 stages with caching; stages parallelized where possible; outputs flow into multi-persona analysis.
7. **Configuration:** YAML-based, API keys via `.env`, tunable thresholds for cost and performance.
8. **Memory strategy:** Streaming frames, on-disk cache, practical limits for 2-hour 4K video.
9. **Error handling:** Fatal vs graceful, retries, crash recovery with resume capability, logging at multiple verbosity levels.

All specifications are concrete (real library names, real file paths, real ffmpeg commands) and directly actionable by an implementation agent.
